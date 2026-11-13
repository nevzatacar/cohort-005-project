# Research: Which email provider, and how does Cadence send email?

Read on 2026-11-13. Every claim links to the primary source it came from. Where a page could not be fetched or did not say something, this doc says so.

## TL;DR

**Use Resend, with plain-text email, through one small `app/server/email.ts` module.**

- **Provider: Resend.** It has the biggest free tier of the paid-API options: 3,000 emails/month, capped at 100/day ([pricing](https://resend.com/pricing)). The SDK is a single `resend.emails.send({ from, to, subject, text })` call ([docs](https://resend.com/docs/send-with-nodejs)). It adds hard-bounced and complained addresses to a suppression list on its own ([suppressions](https://resend.com/docs/dashboard/emails/email-suppressions)), so v1 needs no bounce code.
- **Module:** `sendEmail({ to, subject, text })` checks `EMAIL_TRANSPORT`:
  - `console` (the default in dev and test) logs the message and sends nothing. It adds no dependency.
  - `resend` (production) calls the SDK with `RESEND_API_KEY`.
- **Templates: plain text** for v1. A notification like "your refund was decided" is one sentence and a link. Add React Email later, when someone wants HTML branding. Resend accepts `react:` directly, so that change would be small.
- **Ordering:** write the notification row first. better-sqlite3 is synchronous, so the row is committed by the time the next line runs. Then call `sendEmail()`. If the send fails, log the error and keep the row: the in-app notification is the source of truth and email is best-effort.
- **Skipped for v1:** a Mailpit transport, webhooks, retries, a queue, and storing email status. Add them when a real need shows up (see Open questions).

```ts
// app/server/email.ts (sketch, not yet written)
export async function sendEmail(msg: { to: string; subject: string; text: string }) {
  if (process.env.EMAIL_TRANSPORT !== "resend") return console.log("[email]", msg);
  const { error } = await resend.emails.send({ from: process.env.EMAIL_FROM!, ...msg });
  if (error) console.error("[email] send failed", error); // ponytail: no retry; add a queue if failures matter
}
```

## Comparison

| | Resend | Postmark | AWS SES v2 | nodemailer + SMTP |
|---|---|---|---|---|
| Node SDK | `resend` | `postmark` (no runtime deps, Node ≥18) | `@aws-sdk/client-sesv2` | `nodemailer` (zero runtime deps) |
| Free tier | 3,000/mo, 100/day, 3 domains | 100/mo, no overages | No email-specific free tier; general AWS credits (up to $200, 6-month free plan) | Depends on the SMTP host |
| Cheapest paid plan | $20/mo for 50k (Pro) | $15/mo for 10k (Basic) | Pay as you go | — |
| Overage per 1,000 | $0.90 | $1.80 | $0.10 (+ $0.12/GB attachments) | — |
| Test / sandbox mode | `delivered@`, `bounced@`, `complained@`, `suppressed@resend.dev` | `POSTMARK_API_TEST` token validates the request without sending | Sandbox: verified recipients only, 200/day, 1/sec | Stream transport; works with Mailpit out of the box |
| `react:` accepted directly | Yes | No (render to HTML yourself) | No | No |
| Automatic suppression | Hard bounce + complaint | Hard bounce + complaint (+ unsubscribe) | Hard bounce + complaint (default for accounts created after 2019-11-25) | Depends on the host |
| Setup effort | Low | Low | Highest (IAM, region, sandbox exit with a review of up to 96h) | Low code, but you pick a host |

## Resend

- **API.** From the official Node guide ([source](https://resend.com/docs/send-with-nodejs)):
  ```ts
  import { Resend } from 'resend';
  const resend = new Resend(process.env.RESEND_API_KEY);
  const { data, error } = await resend.emails.send({
    from: 'Acme <onboarding@resend.dev>', to: ['delivered@resend.dev'],
    subject: 'Welcome', react: WelcomeEmail({ name: 'John' }) });
  ```
  `text` and `html` are also accepted ([npm README](https://www.npmjs.com/package/resend)). The SDK returns `{ data, error }` instead of throwing ([source](https://resend.com/docs/send-with-nodejs)).
- **Pricing** ([source](https://resend.com/pricing), read 2026-11-13):
  - Free: 3,000 emails/month, 100/day, 3 domains.
  - Pro: $20/month for 50,000 emails, 10 domains, no daily limit.
  - Overage: $0.90 per 1,000, only once pay-as-you-go is turned on.
- **Test mode.** Sends to `delivered@resend.dev` succeed. Sends to `bounced@resend.dev` get a 550 5.1.1 bounce, `complained@resend.dev` is marked as spam, and `suppressed@resend.dev` comes back as suppressed. Any `+label` works on the first three. Test sends count against your quota ([source](https://resend.com/docs/dashboard/emails/send-test-emails)).
  - `onboarding@resend.dev` is "for testing only"; production needs a verified domain ([source](https://resend.com/docs/send-with-nodejs)).
  - Not verified: which recipients `onboarding@resend.dev` can send to before you verify a domain. The pages I fetched did not say.
- **Templating.** `react:` takes the component called as a function, such as `WelcomeEmail({...})`, per the docs ([source](https://resend.com/docs/send-with-nodejs)). The npm README shows the JSX form instead ([source](https://www.npmjs.com/package/resend)). The README does not say whether `react-email` must also be installed.
- **Bounces and complaints:**
  - A hard bounce or spam complaint puts the address on a suppression list. Later sends to it are marked suppressed and not delivered. The list applies across all of your domains ([source](https://resend.com/docs/dashboard/emails/email-suppressions)).
  - Gmail does not report complaints ([source](https://resend.com/docs/knowledge-base/why-are-my-emails-landing-on-the-suppression-list)).
  - Webhooks cover delivery, bounce and other events ([intro](https://resend.com/docs/dashboard/webhooks/introduction), [event types](https://resend.com/docs/webhooks/event-types)), including a `suppression.added` event.
- **Domain.** Resend recommends sending from a subdomain such as `updates.example.com`, and recommends DMARC once the domain is verified ([source](https://resend.com/docs/dashboard/domains/introduction)). That page does not list the exact SPF and DKIM records; the dashboard shows them when you add a domain.

## Postmark

- **API** ([official libraries page](https://postmarkapp.com/developer/integration/official-libraries)):
  ```ts
  import { ServerClient } from "postmark";
  const client = new ServerClient(process.env.POSTMARK_SERVER_TOKEN!);
  await client.sendEmail({ From: "sender@example.com", To: "receiver@example.com",
    Subject: "Test", TextBody: "Hello from Postmark!", MessageStream: "outbound" });
  ```
  The package has no runtime dependencies and needs Node ≥18. The 4.x line gets no more fixes, so use 5.x ([GitHub](https://github.com/ActiveCampaign/postmark.js)).
- **Pricing** ([source](https://postmarkapp.com/pricing), read 2026-11-13):
  - Free developer plan: 100 emails/month, "No overages allowed".
  - Basic: $15/month for 10,000 emails.
  - Overage: $1.80 per 1,000.
- **Test mode.** Sending the `POSTMARK_API_TEST` server token checks that the request is valid without delivering anything ([source](https://postmarkapp.com/developer/api/overview)).
- **Templating.** Postmark has no `react:` option. You send `TextBody` or `HtmlBody`, so React Email output has to be rendered to HTML first.
- **Bounces and complaints:**
  - Hard bounces, spam complaints and unsubscribes are suppressed ([manual](https://postmarkapp.com/manual), [reactivation](https://postmarkapp.com/support/article/903-an-address-has-been-marked-as-inactive-how-can-i-reactivate-it)).
  - Spam-complaint suppressions cannot be deleted through the API ([Suppressions API](https://postmarkapp.com/developer/api/suppressions-api)).
  - Bounce and spam-complaint webhooks are available ([blog](https://postmarkapp.com/blog/pay-close-attention-to-bounces)).
  - I could not fetch the support article on suppressions (`/support/article/1030-what-are-suppressions`, HTTP 404); the points above come from the other pages linked.
- **Domain.** Verification needs a DKIM TXT record. A custom Return-Path CNAME (`pm-bounces` → `pm.mtasv.net`) is optional but "highly recommend[ed]" ([source](https://postmarkapp.com/support/article/how-do-i-verify-a-domain)).

## AWS SES (v2)

- **API.** I could not read the SDK reference page for `SendEmailCommand`: it renders with JavaScript and came back empty. The shape below comes from the SES v2 API, not a page I could cite:
  ```ts
  import { SESv2Client, SendEmailCommand } from "@aws-sdk/client-sesv2";
  const ses = new SESv2Client({ region: "eu-west-1" });
  await ses.send(new SendEmailCommand({ FromEmailAddress: "a@x.com",
    Destination: { ToAddresses: ["b@y.com"] },
    Content: { Simple: { Subject: { Data: "Hi" }, Body: { Text: { Data: "Hello" } } } } }));
  ```
- **Pricing** ([source](https://aws.amazon.com/ses/pricing/), read 2026-11-13):
  - $0.10 per 1,000 outbound emails, plus $0.12/GB of attachments.
  - No email-specific free tier is listed, only general AWS Free Tier credits ("up to $200"; the free plan lasts 6 months).
- **Sandbox.** New accounts start in a sandbox, separately in each region ([source](https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html)):
  - You can send only to verified addresses or the mailbox simulator.
  - Limits are 200 messages per 24 hours and 1 per second.
  - Leaving the sandbox needs a request; AWS's initial response takes up to 96 hours.
  - The request asks you to confirm you "have a process in place for handling bounce and complaint notifications".
- **Templating.** SES has no React support; you send `Text` and `Html` data.
- **Bounces and complaints:**
  - Accounts created after 2019-11-25 use the account-level suppression list for bounces and complaints by default. Only hard bounces are added, and Gmail does not report complaints ([source](https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html)).
  - Getting bounce and complaint events into the app means wiring up SNS or event destinations.
- **Domain.** Easy DKIM adds a 2048-bit signature once you add DNS records; Route 53 can create them automatically ([source](https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim-easy.html)). A custom MAIL FROM domain is optional.
- **Verdict.** SES is the cheapest at scale, but IAM, regions, the sandbox review and SNS are too much setup for a low-volume app.

## nodemailer + plain SMTP

- **API** ([source](https://nodemailer.com/)):
  ```ts
  const transporter = nodemailer.createTransport({ host: "smtp.example.com", port: 587,
    secure: false, auth: { user: process.env.SMTP_USER, pass: process.env.SMTP_PASS } });
  await transporter.sendMail({ from: '"Team" <team@example.com>', to: "alice@example.com",
    subject: "Hello", text: "Hello world?", html: "<b>Hello world?</b>" });
  ```
  nodemailer has zero runtime dependencies, and its stream transport builds messages without sending them, for tests ([source](https://nodemailer.com/)).
- **Pricing.** nodemailer is free; you still need an SMTP host, which is usually one of the providers above.
- **Local dev.** This is the best fit for Mailpit: point it at `localhost:1025`.
- **Bounces and complaints.** SMTP returns no bounce information to the app; suppression depends on whichever host you use.
- **Verdict.** Choose this only if staying provider-agnostic matters. Otherwise it adds a dependency and still needs a provider behind it.

## Local dev

1. **Default: the `console` transport.** It adds no dependency, nothing leaves the machine, and tests can spy on `console.log` or on `sendEmail` itself. Use this for v1.
2. **Optional: Mailpit**, for when someone wants to see rendered HTML.
   - It catches mail over SMTP, shows it in a web UI, and has a REST API for tests ([source](https://mailpit.axllent.org/)).
   - Install with `brew install mailpit` or Docker. SMTP listens on **1025** and the web UI on **8025** ([source](https://mailpit.axllent.org/docs/install/)).
   - Using it means adding nodemailer as a third `EMAIL_TRANSPORT=mailpit` option. Skip it until HTML templates exist.
3. **Staging smoke test:** set `EMAIL_TRANSPORT=resend` and send to the `delivered@` / `bounced@resend.dev` test addresses ([source](https://resend.com/docs/dashboard/emails/send-test-emails)).

## Bounce handling for v1

- **Do nothing in app code.** Resend suppresses hard bounces and complaints on its own ([source](https://resend.com/docs/dashboard/emails/email-suppressions)), so Cadence cannot keep emailing a dead address.
- The in-app notification still lands, so the user loses nothing.
- **Add later, only when needed:** a webhook route for `email.bounced` / `suppression.added` that flags `users.email_bounced`, so the UI can say "we can't reach your email". This needs signature verification ([source](https://resend.com/docs/dashboard/webhooks/introduction)).

## Open questions

1. Which sending domain or subdomain (for example `notify.<domain>`), and who owns its DNS? Resend recommends a subdomain ([source](https://resend.com/docs/dashboard/domains/introduction)).
2. Is 100/day on the free plan enough? If refunds can come in bursts, budget for Pro ($20/month).
3. Should users be able to opt out per notification type? That would need a preference column. YAGNI until someone asks.
4. Is the email body the same as the notification text, or does it need its own copy? Reusing the text is simplest.
5. Does `react:` need `react-email` installed? Not stated in the README; this only matters once we move off plain text.

## Sources

- Resend: [pricing](https://resend.com/pricing) · [Node guide](https://resend.com/docs/send-with-nodejs) · [test emails](https://resend.com/docs/dashboard/emails/send-test-emails) · [suppressions](https://resend.com/docs/dashboard/emails/email-suppressions) · [suppression KB](https://resend.com/docs/knowledge-base/why-are-my-emails-landing-on-the-suppression-list) · [webhooks](https://resend.com/docs/dashboard/webhooks/introduction) · [event types](https://resend.com/docs/webhooks/event-types) · [domains](https://resend.com/docs/dashboard/domains/introduction) · [npm](https://www.npmjs.com/package/resend)
- Postmark: [pricing](https://postmarkapp.com/pricing) · [API overview](https://postmarkapp.com/developer/api/overview) · [official libraries](https://postmarkapp.com/developer/integration/official-libraries) · [postmark.js](https://github.com/ActiveCampaign/postmark.js) · [manual](https://postmarkapp.com/manual) · [Suppressions API](https://postmarkapp.com/developer/api/suppressions-api) · [verify domain](https://postmarkapp.com/support/article/how-do-i-verify-a-domain)
- AWS SES: [pricing](https://aws.amazon.com/ses/pricing/) · [sandbox](https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html) · [suppression list](https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html) · [Easy DKIM](https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim-easy.html) · [SDK SendEmailCommand](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/sesv2/command/SendEmailCommand/) (could not be read; renders with JavaScript)
- nodemailer: [nodemailer.com](https://nodemailer.com/)
- Mailpit: [home](https://mailpit.axllent.org/) · [install](https://mailpit.axllent.org/docs/install/)
- React Email: [render()](https://react.email/docs/utilities/render)
