# LearnLanka — Requirements Document

## 1. Problem Statement

O/L and A/L students in Sri Lanka need a reliable way to find suitable tutors who match their subject, grade, preferred teaching language, budget, and availability. The current process of finding tutors through personal contacts or informal advertisements can make it difficult to verify tutor quality, compare options, arrange sessions, and manage payments safely. LearnLanka aims to provide a trusted platform where students can discover vetted tutors, book and pay for one-to-one online lessons, attend sessions, and share feedback, while tutors can manage their availability, respond to booking requests, conduct lessons, and receive their earnings.

## 2. Personas

### Persona 1: Student

**Description:**

An O/L or A/L student in Sri Lanka who wants to find a suitable tutor for one-to-one online lessons. The student mainly accesses the platform using an Android mobile device and may prefer to learn in Sinhala, Tamil, or English.

**Goals:**

- Find tutors who teach the required subject and grade.
- Filter tutors by teaching language and price band.
- Review tutor information and available lesson times.
- Book and pay for a one-hour lesson safely.
- Attend the online lesson without a complicated setup.
- Rate the tutor and leave feedback after the lesson.

**Frustrations:**

- Difficulty knowing whether an online tutor is trustworthy.
- Spending time contacting several tutors before finding an available lesson time.
- Unclear prices, cancellation rules, or refund conditions.
- Payment, internet, or video-call problems.
- Receiving a poor-quality lesson without a clear way to provide feedback or request support.

### Persona 2: Tutor

**Description:**

A vetted tutor who teaches O/L or A/L subjects and wants to provide one-to-one online lessons. The tutor needs a simple way to publish available times, manage booking requests, conduct sessions, and receive earnings.

**Goals:**

- Present subjects, grades, teaching languages, rates, qualifications, and experience.
- Publish and update available one-hour lesson slots.
- Accept or decline booking requests.
- Avoid timetable conflicts and duplicate bookings.
- Conduct lessons using the platform’s video service.
- Track completed sessions, commission deductions, and weekly earnings.
- Receive ratings and feedback from students.

**Frustrations:**

- Receiving booking requests for unavailable times.
- Managing schedules and payments through several separate applications.
- Not knowing when payments will be transferred or why money was deducted.
- Handling late cancellations, student no-shows, or technical failures.
- Receiving unfair ratings or comments without a way to report them.

### Persona 3: Operations Admin

**Description:**

A LearnLanka staff member responsible for supporting the platform’s daily operations and resolving problems involving students, tutors, bookings, payments, and external service providers.

**Goals:**

- Review tutor information and manage the tutor-vetting process.
- Monitor bookings, completed sessions, cancellations, payments, commissions, and tutor payouts.
- Support students and tutors when booking, payment, or video-session problems occur.
- Handle disputes, refund requests, reported ratings, and failed payouts.
- Process user consent records and personal-data deletion requests.
- Identify operational problems before they affect many users.

**Frustrations:**

- Incomplete or inconsistent information submitted by tutors.
- Payment or payout failures that require manual investigation.
- Disputes where there is not enough information to determine what happened.
- Unclear cancellation, refund, and no-show policies.
- Managing privacy requests while retaining records required for financial or legal purposes.
- Depending on multiple external services when investigating incidents.

## 3. Functional Requirements

### Student Requirements

1. FR-S1. The platform shall allow a student to search for tutors by subject.
2. FR-S2. The platform shall allow a student to filter tutors by grade level.
3. FR-S3. The platform shall allow a student to filter tutors by teaching language: Sinhala, Tamil, or English.
4. FR-S4. The platform shall allow a student to filter tutors by price band.
5. FR-S5. The platform shall display matching tutors with enough information for the student to compare them, including the tutor’s name, subjects, grades, teaching languages, hourly price, rating, and available lesson times.
6. FR-S6. The platform shall allow a student to select an available one-hour time slot offered by a tutor.
7. FR-S7. The platform shall allow a student to submit a booking request for the selected tutor and time slot.
8. FR-S8. The platform shall allow a student to pay for a booking using either a card or eZ Cash.
9. FR-S9. The platform shall show the current status of each booking, such as pending, accepted, declined, cancelled, or completed.
10. FR-S10. The platform shall notify the student when a tutor accepts, declines, or cancels a booking.
11. FR-S11. The platform shall provide the student with access to the online video session at the scheduled time.
12. FR-S12. The platform shall allow a student to give the tutor a rating from 1 to 5 stars after the session is completed.
13. FR-S13. The platform shall allow a student to leave a one-line comment about the tutor after the session is completed.
14. FR-S14. The platform shall allow a student to select Sinhala, Tamil, or English as the interface language.
15. FR-S15. The platform shall allow a student to submit a request to delete their personal data.

### Tutor Requirements

1. FR-T1. The platform shall allow a tutor to publish available one-hour lesson slots.
2. FR-T2. The platform shall allow a tutor to update or remove an availability slot that has not already been booked.
3. FR-T3. The platform shall allow a tutor to view pending booking requests.
4. FR-T4. The platform shall allow a tutor to accept or decline a booking request.
5. FR-T5. The platform shall prevent an accepted lesson slot from being booked by another student for the same time.
6. FR-T6. The platform shall allow a tutor to cancel an accepted booking only when at least 12 hours remain before the scheduled start time.
7. FR-T7. The platform shall notify the tutor when a new booking request is received.
8. FR-T8. The platform shall provide the tutor with access to the online video session at the scheduled time.
9. FR-T9. The platform shall allow a tutor to give the student a rating from 1 to 5 stars after the session is completed.
10. FR-T10. The platform shall allow a tutor to leave a one-line comment about the student after the session is completed.
11. FR-T11. The platform shall allow a tutor to view completed sessions, the total session fee, the 15% platform commission, and the remaining amount payable to the tutor.
12. FR-T12. The platform shall show the tutor the status of each weekly payout.
13. FR-T13. The platform shall allow a tutor to select Sinhala, Tamil, or English as the interface language.
14. FR-T14. The platform shall allow a tutor to submit a request to delete their personal data.

### Operations Admin Requirements

1. FR-A1. The platform shall allow an Operations Admin to review tutor applications and approve or reject tutors before they become visible to students.
2. FR-A2. The platform shall allow an Operations Admin to view students, tutors, bookings, completed sessions, cancellations, payments, commissions, and payout statuses.
3. FR-A3. The platform shall allow an Operations Admin to view payment failures and tutor payout failures for investigation.
4. FR-A4. The platform shall allow an Operations Admin to review personal-data deletion requests and record whether each request is pending, approved, completed, or rejected for a valid legal reason.
5. FR-A5. The platform shall allow an Operations Admin to review reported ratings or comments and take appropriate moderation action.
6. FR-A6. The platform shall allow an Operations Admin to view booking history and session status when investigating a complaint or dispute.
7. FR-A7. The platform shall allow an Operations Admin to identify tutor payments that are due in the next weekly payout cycle.

### Platform and Business Rules

1. FR-B1. The platform shall calculate a 15% commission for every session marked as completed.
2. FR-B2. The platform shall calculate the tutor’s earnings as 85% of the completed session fee before any separately defined fees or deductions.
3. FR-B3. The platform shall include eligible completed-session earnings in the tutor’s weekly bank payout.
4. FR-B4. The platform shall send tutor payouts through Sampath Vishwa.
5. FR-B5. The platform shall process student payments through PayHere.
6. FR-B6. The platform shall create or provide access to video sessions through the selected third-party video provider.
7. FR-B7. The platform shall capture the user’s consent before collecting or processing personal data that requires consent.
8. FR-B8. The platform shall support Sinhala, Tamil, and English for all user-interface text from launch.

## 4. Non-Functional Requirements

| Category | Metric | Target | How we will measure it |
|---|---|---|---|
| Performance | Tutor search response time measured at the 95th percentile from a Sri Lankan ISP | Less than 800 ms | Azure Application Insights and scheduled performance tests from a Sri Lankan network |
| Availability | Percentage of time the booking endpoint successfully accepts valid requests | At least 99.5% per calendar month | Azure Monitor and automated synthetic requests |
| Concurrency | Number of video sessions active at the same time | At least 200 simultaneous sessions during the first six months | Load testing and the third-party video provider’s dashboard |
| Payment security | Number of card or eZ Cash credentials stored on LearnLanka servers | Zero | Database inspection, log review, and payment-integration security testing |
| Payment compliance | Percentage of card and eZ Cash payments processed through the PCI-DSS-compliant PayHere gateway | 100% | Payment integration logs and security review |
| Localization | Percentage of user-facing interface strings available in Sinhala, Tamil, and English | 100% from launch | UI string inventory and localization acceptance testing |
| Privacy | Percentage of applicable personal-data processing activities with recorded user consent | 100% | Consent-record review and privacy acceptance testing |

## 5. Assumptions

The following assumptions have been made because the brief does not provide enough detail. These assumptions must be confirmed with the founders before development begins.

1. Students, tutors, and Operations Admins will each have separate user accounts and role-based access to the platform.
2. Students and tutors must register and log in before they can make or manage bookings.
3. Since many O/L and A/L students may be under 18, the platform may need to capture parent or guardian consent where legally required.
4. Tutors must complete a vetting process before their profiles become visible in student search results.
5. The Operations Admin is responsible for reviewing tutor applications and approving or rejecting tutors.
6. Tutors are responsible for setting their own hourly session price, subject to any limits later defined by LearnLanka.
7. A tutor’s availability slot remains temporarily unavailable to other students while a booking request is pending.
8. A booking is not treated as confirmed until the tutor accepts it.
9. A session is considered completed only after the scheduled session time has ended and the platform has recorded a successful session status.
10. Ratings and comments can only be submitted after a session has been marked as completed.
11. Weekly tutor payouts include only earnings from sessions that were completed before the payout cut-off time.
12. The first version will be mobile-first and will support common Android web browsers, but a separate native Android application is not assumed unless confirmed.
13. The Operations Admin will be able to investigate booking, payment, payout, rating, and privacy-related issues.
14. Personal-data deletion requests may not immediately remove records that must be retained for legal, financial, security, or audit purposes.

These assumptions give the team a workable starting point, but the higher-risk ones, especially payment timing, cancellation rules, underage consent, tutor vetting, and session completion, should also appear in the Ambiguity Hunt Log.

## 6. Out of Scope

The following items will not be included in the first version of LearnLanka unless the founders approve them as additional requirements.

1. Building an in-house video-calling system. Video sessions will be handled by Daily.co or 100ms.
2. Group classes involving multiple students in the same session. The first version will support only one-to-one sessions.
3. Pre-recorded lessons, downloadable study materials, quizzes, and full online-course management.
4. Face-to-face or physical tuition-class bookings. LearnLanka will focus on online sessions.
5. Payment methods other than card and eZ Cash through PayHere.
6. Payout methods other than weekly bank transfers through Sampath Vishwa.
7. Automatic tutor recommendations using artificial intelligence or advanced matching algorithms. Students will search and filter tutors using the available criteria.
8. International tutors, foreign currencies, and educational programmes outside Sri Lankan O/L and A/L subjects.
9. Payroll, tax calculation, accounting, or employment management for tutors beyond recording commissions and processing eligible payouts.
10. Real-time chat, social networking, community forums, or communication between users outside the booking and session process.
