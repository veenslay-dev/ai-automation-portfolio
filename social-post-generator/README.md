# Social Post Generator (Human-in-the-Loop Approval)

## Overview
A Telegram-driven content workflow. Send it a post idea and it generates a caption 
and an image, sends both back for review, then waits. The reviewer approves or rejects 
through a small web form. On rejection, the written feedback is passed to the AI, which 
produces a revised caption and image. The cycle repeats until the post is approved.

## How It Works
1. **Telegram Trigger** receives the post idea
2. **AI Agent** (Gemini) returns a caption and an image description in a fixed format
3. **Edit Fields** splits that text into `caption` and `imagePrompt` fields
4. **HTTP Request** downloads the generated image as a binary file
5. **Send Photo** delivers the image and caption on Telegram
6. A second message carries the approval form link (`$execution.resumeFormUrl`)
7. **Wait (On Form Submitted)** pauses the workflow until the form is submitted 
   (Decision dropdown + optional Feedback text)
8. **Switch** routes on the decision:
   - **Approve**: workflow ends (platform posting is the next planned step)
   - **Reject**: a second AI Agent receives the previous caption, previous image 
     description and the feedback, then produces a revised version that flows 
     back into step 3

## Tech Stack
n8n · Google Gemini API · Telegram Bot API · pollinations.ai (image generation)

## Challenges & Solutions
- **Free-text feedback through a link was not possible**: the first approach used a 
  webhook resume URL with `?decision=approve`, which only supports fixed values. 
  Switched the Wait node to "On Form Submitted" so the reviewer can choose a decision 
  and write feedback.
- **"Invalid token" on the approval link**: n8n's resume URL already contains 
  `?signature=...`, and appending `?decision=approve` produced two `?` in one URL. 
  Fixed by appending with `&` instead.
- **Reject loop only resent the form link**: the revision AI's output was connected 
  to the wrong node, so the image and caption were never regenerated. Reconnected it 
  to the Edit Fields node at the start of the generation chain.
- **Testing confusion**: "Execute step" inside a node reuses old sample data, so pause 
  and resume behavior only showed up in full live runs checked from the Executions tab.
- **External API instability**: the free image API occasionally returned an internal 
  server error, unrelated to the workflow logic.

## Result
A working approve/reject loop with feedback-driven regeneration. One post rejected three 
times and approved on the fourth pass produces four Wait pauses, four Switch routings 
and three revision runs.

## Next Steps
- Post to a social platform on Approve
- Add a wait time limit so unattended runs do not stay paused forever
