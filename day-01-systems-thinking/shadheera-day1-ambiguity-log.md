# LearnLanka - Ambiguity Hunt Log

## Brief Reference

The following statements are taken from the LearnLanka business brief:

“Students must be able to search for tutors by subject, grade, language (Sinhala, Tamil, English), and price band.”

“Students must be able to book a 1-hour session with a tutor and pay via card or eZ Cash.”

“Tutors must be able to publish availability slots, accept or decline bookings, and cancel with at least 12 hours notice.”

“The platform must charge a 15% commission on every completed session and pay tutors weekly via bank transfer.”

“Both parties must be able to rate each other (1-5 stars) and leave a one-line comment after the session.”

“Tutor search results: returned in under 800 ms at the 95th percentile from a Sri Lankan ISP.”

“Availability: 99.5% monthly uptime, measured against the booking endpoint.”

“Concurrent sessions: support 200 simultaneous video sessions in the first 6 months.”

“Privacy: comply with Sri Lanka Personal Data Protection Act 2022 - consent capture, deletion request flow.”

“Video calling will be outsourced to a third-party (Daily.co or 100ms) - not built in-house.”

“All UI strings must support Sinhala, Tamil, and English from launch.”

“LearnLanka ... connects O/L and A/L students with vetted tutors.”

“Mobile-first product - at least 80% of usage is expected on Android devices.”

These statements define the general expectations, but several business rules and measurable details are not yet clear enough for implementation and testing.

## Findings

| #  | Quote from brief                                                                                             | Why it is ambiguous                                                                                                                                                                   | Clarification question                                                                                                                      | Priority |
| -- | ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| 1  | “Students must be able to book a 1-hour session with a tutor and pay via card or eZ Cash.”                   | It does not state whether payment happens before or after the tutor accepts the booking. This affects booking status, payment processing and refunds.                                 | Should the student's payment be authorised or captured before the tutor accepts the booking, or only after acceptance?                      | H        |
| 2  | “Tutors must be able to ... accept or decline bookings.”                                                     | The brief does not explain what happens to a student's payment when a tutor declines an already-paid booking.                                                                         | If a tutor declines a paid booking, should the student receive a full automatic refund, and within what period?                             | H        |
| 3  | “Tutors must be able to ... cancel with at least 12 hours notice.”                                           | It is unclear whether tutors are completely prevented from cancelling within 12 hours or whether an admin override, penalty or emergency process exists.                              | What should happen if a tutor needs to cancel less than 12 hours before the session?                                                        | H        |
| 4  | “Tutors must be able to ... cancel with at least 12 hours notice.”                                           | A cancellation rule is provided for tutors, but no cancellation rule is provided for students.                                                                                        | Can students cancel or reschedule a booking, and what notice period, refund and penalty rules should apply?                                 | H        |
| 5  | “The platform must charge a 15% commission on every completed session.”                                      | The word “completed” is not defined. The platform needs to know whether completion is based on time, video attendance, tutor confirmation, student confirmation or an admin decision. | What exact event or evidence should cause a session to be marked as completed?                                                              | H        |
| 6  | “The platform must ... pay tutors weekly via bank transfer.”                                                 | The payout day, cut-off time, minimum payout amount, failed-transfer handling and possible transfer fees are not specified.                                                           | On which day and at what cut-off time are weekly payouts calculated, and what happens when a bank transfer fails?                           | H        |
| 7  | “Both parties must be able to rate each other (1-5 stars) and leave a one-line comment after the session.”   | “One-line comment” has no measurable length. The brief also does not say whether reviews can be edited, deleted, reported or submitted more than once.                                | What is the maximum comment length, can a review be edited, and may each party submit only one review per session?                          | M        |
| 8  | “LearnLanka ... connects O/L and A/L students with vetted tutors.”                                           | The meaning of “vetted” is not defined. The required documents, qualifications, checks and approval authority are unknown.                                                            | What information and evidence must a tutor provide, and what criteria must be satisfied before approval?                                    | H        |
| 9  | “Students must be able to search for tutors by ... price band.”                                              | The price-band ranges and their boundary rules are not provided. It is also unclear whether tutors freely set prices or LearnLanka defines permitted rates.                           | What are the exact price bands, are the boundaries inclusive, and who controls a tutor's hourly rate?                                       | M        |
| 10 | “Tutor search results: returned in under 800 ms at the 95th percentile from a Sri Lankan ISP.”               | It is unclear whether the 800 ms covers only the server response or the entire screen-loading experience. The expected data volume and user load are also missing.                    | Does the 800 ms target apply to the search API response only or to complete result rendering, and under what expected load and data volume? | H        |
| 11 | “Availability: 99.5% monthly uptime, measured against the booking endpoint.”                                 | The brief does not define which responses count as available, whether planned maintenance is excluded or from which location the endpoint is monitored.                               | How should uptime be calculated, which responses count as successful, and is planned maintenance excluded?                                  | H        |
| 12 | “Concurrent sessions: support 200 simultaneous video sessions in the first 6 months.”                        | It is unclear whether this means 200 video rooms or 200 individual participants. The expected session duration and provider capacity plan are also not stated.                        | Does “200 simultaneous video sessions” mean 200 active rooms with one student and one tutor in each room?                                   | H        |
| 13 | “Privacy: comply with Sri Lanka Personal Data Protection Act 2022 - consent capture, deletion request flow.” | Many users may be minors, but parent or guardian consent is not addressed. The deletion deadline and legally retained data are also unclear.                                          | What consent is required for users under 18, how quickly must deletion requests be completed, and which records must be retained?           | H        |
| 14 | “Video calling will be outsourced to a third-party (Daily.co or 100ms).”                                     | The provider has not been selected. The two providers may have different integrations, pricing, security controls and monitoring capabilities.                                        | Which video provider will be used for the first version, and which required features must the integration support?                          | H        |
| 15 | “All UI strings must support Sinhala, Tamil, and English from launch.”                                       | It is unclear whether this includes system notifications, emails, SMS messages, user-generated comments and external-provider screens.                                                | Which content must be translated at launch: application screens, validation messages, notifications, emails, SMS and provider-hosted pages? | M        |
| 16 | “Mobile-first product - at least 80% of usage is expected on Android devices.”                               | The brief does not say whether the product is a responsive website, PWA or native Android application. Supported Android versions and browsers are also undefined.                    | Is the first version a responsive web application, PWA or native application, and which Android versions and browsers must be supported?    | H        |

## Results Summary

| Metric                          | Target | Achieved |
| ------------------------------- | ------ | -------- |
| Items found                     | 10+    | 16       |
| High-priority items             | 3+     | 13       |
| Items convertible to test cases | 5+     | 14       |

## Items Convertible to Test Cases

The following ambiguities can directly become acceptance criteria or test cases after the founders provide answers:

1. Whether payment happens before or after tutor acceptance.
2. Refund result and completion time after tutor decline.
3. Tutor cancellation behaviour inside the 12-hour limit.
4. Student cancellation and refund rules.
5. The exact condition that marks a session as completed.
6. Weekly payout cut-off and failed-payout behaviour.
7. Rating and comment length and submission limits.
8. Tutor-vetting criteria.
9. Exact price-band boundaries.
10. Search-performance measurement boundary.
11. Booking-endpoint availability calculation.
12. Meaning of 200 simultaneous sessions.
13. Privacy-request completion time.
14. Supported devices, Android versions and browsers.

## Top 3 Questions to Ask the Founders

1. When should the student's payment be captured, and what refund rules apply when a tutor declines or cancels a paid booking?
2. What exact event or evidence should mark a session as completed for commission, ratings and weekly tutor payouts?
3. What are the complete cancellation, rescheduling and no-show rules for students and tutors?

## Reflection

### What kind of ambiguity tripped me up most?

The most difficult ambiguities were the ones where one requirement affected several other parts of the system. For example, the brief says that students pay for sessions while tutors can later accept, decline or cancel bookings. Without knowing the payment timing and refund policy, it is difficult to complete the booking, payment, notification and dispute requirements. These gaps may look small in one sentence, but they affect multiple user journeys and external integrations.

### Which question is most likely to change the architecture if answered?

The question about when payment is captured is most likely to change the architecture. Capturing payment before tutor acceptance would require pending-payment handling, slot reservation, automatic refunds and transaction-status tracking. Capturing payment after acceptance would require the student to return and complete payment within a defined time while the selected slot remains reserved. These two approaches create different workflows, booking states, scheduled processes and failure scenarios.

### Why is ambiguity expensive?

Ambiguity is expensive because different team members may make different assumptions and build incompatible parts of the system. Developers may implement one payment flow while testers expect another, and the business may only discover the mismatch after development has already taken place. Asking specific questions early reduces rework and helps the team create requirements that are estimable, testable and aligned with stakeholder expectations.
