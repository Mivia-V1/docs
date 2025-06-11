
# Product Requirements Document: MiVia Home Page

**Filename:** prd-mivia-home-page.md

---

## 1. Introduction / Overview

The **MiVia Home Page** serves as the main landing screen for the MiVia iOS app. It provides users with immediate access to key app features, including quick daily check-ins, inspirational content, status notifications, and navigation across the app’s core areas. Its primary goal is to engage users at launch, guide them intuitively through the app, and establish a supportive and motivating environment that embodies MiVia’s focus on psychology-based personal growth.

---

## 2. Goals

- Deliver a visually appealing and welcoming home interface using the latest SwiftUI techniques.
- Display critical daily information (e.g., inspirational quote, check-in prompt) to encourage consistent app usage and psychological engagement.
- Present an at-a-glance summary of unread messages and quick navigation options.
- Ensure the home page acts as a seamless entry-point into the four main life pillars and related features.

---

## 3. User Stories

1. **As a returning user**, I want to see the app’s logo and a brief welcome message so I feel recognized and motivated.
2. **As a user**, I want to quickly log my daily check-in rating from the home screen so that my progress is easily tracked.
3. **As a user**, I want to read a daily inspirational quote on the home page so I feel encouraged to continue my growth journey.
4. **As a user**, I want to immediately see if I have unread messages or notifications so I never miss important updates or coaching.
5. **As a user**, I want a clear and consistent navigation bar so I can easily move between different sections of the app.

---

## 4. Functional Requirements

1. **Header with Logo**
    - Display the MiVia logo prominently at the top of the home page.
    - Optionally include the user’s name (e.g., “Welcome back, [Name]!”) if available from profile data.
2. **Welcome Banner**
    - Show a visually appealing banner with a daily inspirational quote fetched dynamically (from Supabase or local quotes data).
    - The quote should refresh each day.
3. **Daily Check-in Rating**
    - Present a simple interface element (e.g., mood slider, emoji selector, rating bar) for the user to input their current feeling/state.
    - Store the check-in response in Supabase, tagged with the datetime and user ID.
    - If user has already checked in for the day, show their current check-in and allow editing until midnight.
4. **Unread Message Indicator**
    - Display a clear icon or badge if there are unread messages in the user’s inbox (pulled from existing messaging/notifications module).
    - Tapping the indicator navigates to the Messages screen.
5. **Bottom Navigation Bar**
    - Provide a persistent navigation bar at the bottom using SwiftUI’s TabView.
    - Tabs should include: Home, Purpose, Health, Emotion, Connection, and Messages/Notifications.
    - Highlight the current section.
6. **Modern, Welcoming UI**
    - Use latest SwiftUI Views, animations, gradients, and colors as aligned with MiVia branding.
    - Ensure all UI components are visually accessible and support Dark Mode.
    - Apply padding, corner radii, shadows, and other style details to create warmth and approachability.

---

## 5. Non-Goals (Out of Scope)

- Full implementation of the four pillars' inner pages (Navigation only, not detailed content).
- In-depth settings, user onboarding, or profile editing features.
- Detailed analytics or progress visualizations on the Home page.
- Push notification configuration or deep notification management (indicator only).
- Support for Android or other platforms.

---

## 6. Design Considerations

- **UI/UX:** Prioritize clarity, warmth, and positivity. Reference Figma mockups [insert link once available]. Use large, clear typography for quotes and check-ins.
- **Responsive Layout:** Support all current iPhone screen sizes (AutoLayout via SwiftUI).
- **Transitions:** Subtle animations when switching tabs, displaying the check-in modal, or loading a new quote.
- **Accessibility:** Ensure all actionable elements have proper accessibility labels.

---

## 7. Technical Considerations

- **SwiftUI 3+**: Use latest capabilities, including async image loading, view builders, and modifiers.
- **Supabase Integration**: Fetch/store check-in data and daily quotes where applicable; ensure APIs are called asynchronously.
- **Auth Dependency**: Check for user authentication before displaying personalized data.
- **State Management**: Use @State, @ObservedObject, or @EnvironmentObject (with best practice) for data handling.
- **Localization**: Prepare for potential multi-language support in UI text.
- **Dark Mode**: Full support for both light and dark system themes.

---

## 8. Success Metrics

- ≥80% of returning users interact with the check-in or quote section within 7 days of app use.
- Increase daily active user sessions by 15% post-launch.
- >90% accuracy in unread message indicator syncing with actual messages.
- Positive user feedback (4+ stars on design via in-app survey).
- Reduce bounce rate from home page (users immediately exiting app) by 10%.

---

## 9. Open Questions

- What is the source of the daily inspirational quotes (Supabase table or third-party API)?
- Should the check-in type (e.g., emoji, slider, text) be configurable or is a specific format required?
- Are there specific navigation icons or branding guidelines to follow for the logo/banner?
- How should unread message status be computed (e.g., system notifications or only chat messages)?
- Who is responsible for maintaining and updating quotes or motivational content?

---

**Prepared for:**  
Junior Developers working on MiVia iOS app (SwiftUI + Supabase)

**Contact:**  
[Product Owner Name / Slack Contact]

---
```
