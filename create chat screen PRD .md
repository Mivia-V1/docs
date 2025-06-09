```markdown
# Product Requirements Document: Create Chat Screen

Filename: prd-create-chat-screen.md

---

## Introduction / Overview

The **Create Chat Screen** feature provides users with a dedicated interface to interact with MiVia’s Coaching AI. By mimicking the familiar WhatsApp chat style, this screen aims to facilitate seamless, real-time communication between users and the AI coach. The goal is to increase engagement and support users' behavior change journeys across MiVia's four life pillars: Purpose, Health, Emotion, and Connection. The chat history will be timestamped, helping users reflect on previous interactions and track their personal growth over time.

---

## Goals

1. **Enable Real-Time AI Coaching Interaction:** Allow users to send and receive messages with the Coaching AI in a conversational format.
2. **User-Friendly and Familiar UI:** Mimic WhatsApp’s chat interface to lower learning curves and increase user comfort.
3. **Provide Chat History:** Display recent chat history, with each message clearly time- and date-stamped.
4. **Support Behavior Change:** Help users feel supported and motivated through easy access to habit- and psychology-based guidance from the AI coach.

---

## User Stories

- **As a user,** I want to send messages to my Coaching AI so that I can ask for advice, motivation, or support.
- **As a user,** I want to see a history of my interactions with the AI so that I can reflect on my progress and previous guidance.
- **As a user,** I want the chat interface to feel familiar (like WhatsApp), so I can comfortably use the feature without learning a new format.
- **As a user,** I want each message (from myself and the AI) to show the time and date, so I can track when conversations happened.

---

## Functional Requirements

1. **Chat Interface**
   - 1.1. Implement a chat screen using SwiftUI, styled similarly to WhatsApp (left/right message bubbles, avatars optional).
   - 1.2. Input text box for user messages, with ‘Send’ button.

2. **Message Handling**
   - 2.1. Allow users to type and send messages to the AI.
   - 2.2. Display AI responses in real time.
   - 2.3. Show both user and AI messages in the chat thread.
   - 2.4. Time and date-stamp each message in the thread.
   - 2.5. Scroll to the newest message as messages are exchanged.

3. **Chat History**
   - 3.1. Retrieve and display recent chat history for the currently authenticated user.
   - 3.2. Store chat messages (user and AI) in Supabase, including timestamps.

4. **AI Integration**
   - 4.1. Integrate with the Coaching AI backend (ensure request/response cycle is robust).

5. **Authentication**
   - 5.1. Ensure chat history is user-specific and protected by the existing Auth module.

---

## Non-Goals (Out of Scope)

- **Media Attachments:** No support for sending/receiving media (photos, videos, audio) at launch.
- **Group Chats:** Only individual AI-user chat is supported; no user-to-user or group chats.
- **Push Notifications:** No push notifications for new AI responses in this version.
- **Advanced Formatting:** No rich text, emoji reactions, or message editing.

---

## Design Considerations

- Use SwiftUI components where possible to align with the app’s architecture.
- Chat bubbles should match WhatsApp’s general appearance for usability.
- Each message must display a readable time and date (e.g., "14:32, Jun 12, 2024").
- Input area should not obscure messages due to keyboard.
- Consider accessibility features (font scaling, color contrast, VoiceOver compatibility).

---

## Technical Considerations

- Integrate with Supabase for message storage, retrieval, and user authentication.
  - Ensure all messages are properly associated with the user’s unique ID.
- Interface with existing AI backend for generating responses.
- Optimize for performance to handle real-time message exchange.
- Handle error states gracefully (e.g., failed message delivery, delayed AI response).
- Modularize code for future enhancements (e.g., media messages, group chat).

---

## Success Metrics

- **Chat Feature Engagement:** At least 40% of active users engage with the chat feature weekly within 1 month of launch.
- **User Satisfaction:** User feedback indicates at least 80% positive rating for chat usability and effectiveness (collected via in-app survey).
- **Support Tickets:** Fewer than 5% of total support tickets relate to chat feature bugs or usability issues within the first 2 months.
- **Behavior Change Engagement:** 20% increase in users completing at least one behavior change action after AI chat usage.

---

## Open Questions

- What is the maximum desired length for chat history (e.g., last 100 messages or all time)?
- Should users be able to delete individual messages or the entire conversation?
- Will avatars or user profile images be shown in the chat bubbles?
- Should there be a typing indicator when the AI is generating a response?
- Are there compliance or privacy requirements for storing AI-user chat data (e.g., GDPR)?

---

```
**End of PRD for "Create Chat Screen"**
```
