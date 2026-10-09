# IT-Support-Helper

An AI first-line IT support bot built with n8n and Gemini. It answers an employee's IT problem using only a given knowledge base, and logs a ticket to Google Sheets when it can't solve the problem or when the issue needs IT staff.

> **This is a prototype.** The knowledge base is fictional (IT scenarios for an imaginary photo shop), and the tool's effect on support request volume has not been measured.

## How it works

![Workflow](docs/it_helper_workflow.png.png)

1. **Chat Trigger**: the user types their problem into the chat.
2. **Gemini**: the model answers based on the knowledge base in the system prompt and returns JSON (`confident`, `needs_ticket`, `category`, `priority`, `answer`, `summary`).
3. **Parse JSON**: a Code node strips any code fences and turns the model's text output into structured data.
4. **IF `needs_ticket`**:
   - **true** (e.g. password reset, phishing, POS down, or the bot doesn't know): the user sees the instructions and a ticket is logged to Sheets with status "Avoin" (open).
   - **false**: the user sees the solution and is asked "Did this solve the problem?"
5. **Solved?**
   - Yes: a row is logged with status "Ratkaistu botilla" (solved by bot).
   - No: a row is logged with status "Avoin" and the user is told a ticket has been created.

Logging solved vs. open lets you see which problems recur and what to add to the knowledge base.

## Design decisions

- **Answer only from the knowledge base.** The model must not guess. If no answer is found it returns `confident=false` and a ticket is created.
- **`confident` and `needs_ticket` are separate.** The first says whether the model knows the answer, the second whether a human needs to act. For a password reset the model knows the procedure (IT handles it), but a ticket is still required.
- **Guardrails in the prompt.** The bot never asks for passwords, never handles customer personal data, and never instructs anyone to delete customer photos or orders.
- **Escalation.** Phishing, lost image data, POS/payment terminal failure and multi-workstation outages always create a high-priority ticket.
- **Retry On Fail** on the Gemini node, since the model API can be temporarily overloaded.

## Setup

1. Import `workflow/it-support-helper.json` into n8n (**Import from File**).
2. Create your own credentials:
   - Gemini API key (Google AI Studio)
   - Google Sheets OAuth2 (enable the Sheets API and Drive API in Google Cloud)
3. Create a Google Sheet with this header row:
   `pvm`, `aihe`, `kategoria`, `prioriteetti`, `tila`, `alkuperainen_viesti`
   (date, subject, category, priority, status, original message)
4. Select the sheet in the three Append Row nodes.
5. In the Chat Trigger, set Response Mode to **Using Response Nodes**.
6. Open the chat from the canvas (**Open chat**) and try it out.

The system prompt is in [`prompts/system-prompt.md`](prompts/system-prompt.md). The prompt, ticket statuses and sheet columns are in Finnish.

## Test cases

| Input                           | Expected result                                                |
| ------------------------------- | -------------------------------------------------------------- |
| Paper jam in the minilab        | Instructions, "solved?" prompt, row logged based on the answer |
| I forgot my password            | Ticket created directly, bot does not ask for the password     |
| Suspicious email with a link    | Ticket, high priority, advice not to click the link            |
| What is the capital of Finland? | Bot doesn't answer, ticket created                             |

## Limitations

- The knowledge base is hand-written and fictional. In real use it would need to be built from the actual environment's devices and software, and maintained.
- The reporter is not identified. In production the bot could live in a tool staff already use (e.g. Teams or Slack), so the user comes from the trigger automatically.
- Tickets go to Google Sheets, not a real ticketing system.
- Answer quality has not been evaluated at scale; testing was done manually on a handful of cases.
- Do not enter customer data or passwords into the bot.

## Possible next steps

- Teams or Slack integration with user identification
- Tickets in a real ticketing system
- Retrieval from documentation instead of a fixed prompt
- Reporting on resolution rate and recurring issues from the Sheets data

## Stack

n8n (self-hosted, Docker), Google Gemini API, Google Sheets
