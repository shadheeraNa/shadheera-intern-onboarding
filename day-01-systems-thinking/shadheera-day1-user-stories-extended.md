# LearnLanka — User Story Set v0.1

## Story 1: Search Tutors by Subject, Grade, Language and Price Band

### As a student

I want to search for tutors by subject, grade, teaching language and price band
So that I can find tutors who match my learning needs and budget.

### Acceptance Criteria

- Given a student is on the tutor search page, when the student selects a subject and performs a search, then only tutors who teach that subject are displayed.
- Given the student selects a grade, teaching language and price band, when the search is performed, then every displayed tutor matches all selected filters.
- Given matching tutors are found, when the results are displayed, then each result shows the tutor's name, subjects, supported grades, teaching languages, hourly price, rating and available lesson times.
- Given no tutor matches the selected filters, when the search is performed, then the platform displays a message explaining that no matching tutors were found.

### INVEST Self-Check

- YES - Independent - The tutor-search capability can be developed and tested without depending on booking, payment or video-session features.
- YES - Negotiable - The story does not prescribe the search-screen layout, filter controls or technical implementation.
- YES - Valuable - It allows students to find a useful shortlist of tutors who match their learning needs and budget.
- YES - Estimable - The supported filters, displayed information and expected outcomes are clearly defined.
- YES - Small - The story covers one cohesive capability: filtering and displaying matching tutors.
- YES - Testable - Every acceptance criterion has an observable pass-or-fail outcome.

## Story 2: Fast Tutor Search Response Time

### As a student

I want tutor search results to load quickly
So that I can compare tutors without unnecessary delays.

### Acceptance Criteria

- Given valid tutor searches are performed through a Sri Lankan ISP, when response times are measured, then at least 95% of search responses are returned in less than 800 milliseconds.
- Given searches using subject, grade, teaching language and price-band filters are tested under the agreed initial operating conditions, when the 95th-percentile response time is calculated, then it remains below 800 milliseconds.

### INVEST Self-Check

- YES - Independent - Search performance can be measured separately from booking, payment and video-session capabilities.
- YES - Negotiable - The required outcome is defined without prescribing caching, database or infrastructure techniques.
- YES - Valuable - Quick search results reduce delays while students are trying to find and compare tutors.
- YES - Estimable - The response-time target, percentile and network context are clearly specified.
- YES - Small - The story focuses on one non-functional concern: tutor-search response time.
- YES - Testable - The response-time target is numeric and can be verified through repeatable performance testing.

## Story 3: Request an Available Lesson Slot

### As a student

I want to request an available one-hour lesson slot
So that I can reserve a suitable time with a tutor.

### Acceptance Criteria

- Given a student is viewing a tutor's available lesson times, when the student selects an available slot, then the platform allows the student to submit a booking request.
- Given the booking request is successfully submitted, when the student views the booking, then its status is displayed as pending.
- Given a booking request is pending for a time slot, when another student views the same tutor's availability, then that slot is not displayed as available.
- Given the selected slot is no longer available, when the student attempts to submit the booking request, then the platform prevents the request and asks the student to select another slot.

### INVEST Self-Check

- YES - Independent - The booking-request capability can be developed and tested using available tutor slots without depending on payment or video-session features.
- YES - Negotiable - The story does not specify how the available slots must be displayed or how the request screen should be designed.
- YES - Valuable - It allows the student to request a suitable lesson time and prevents conflicting requests for the same slot.
- YES - Estimable - The booking states and expected slot behaviour are clearly described.
- YES - Small - The story covers one capability: requesting and temporarily reserving an available slot.
- YES - Testable - Each criterion produces a visible booking status or availability result.

## Story 4: Pay for a Booking

### As a student

I want to pay for a booking using a card or eZ Cash
So that I can complete the required payment for my lesson.

### Acceptance Criteria

- Given a booking is ready for payment, when the student chooses card or eZ Cash, then the payment is processed through PayHere.
- Given PayHere confirms that the payment was successful, when LearnLanka receives the result, then the booking records the payment as successful.
- Given PayHere reports that the payment failed or was cancelled, when LearnLanka receives the result, then the booking is not marked as paid and the student is informed.
- Given the student enters card or eZ Cash payment details, when the payment is processed, then those payment credentials are not stored on LearnLanka servers.

### INVEST Self-Check

- YES - Independent - The payment flow can be developed and tested using a booking that is already ready for payment.
- YES - Negotiable - The story defines the payment methods and outcome but does not prescribe the payment-page design or integration structure.
- YES - Valuable - It enables students to pay securely using one of the supported payment methods.
- YES - Estimable - PayHere, the supported payment methods and the expected success and failure outcomes are identified.
- YES - Small - The story focuses only on processing and recording one booking payment.
- YES - Testable - Successful, failed and cancelled payment results can each be verified, including the requirement not to store payment credentials.

## Story 5: View Booking Status

### As a student

I want to view the current status of my booking
So that I know whether my lesson is pending, accepted, declined, cancelled or completed.

### Acceptance Criteria

- Given a student has submitted a booking request, when the student views their bookings, then the current status of each booking is displayed.
- Given a tutor accepts or declines a booking, when the student views that booking, then the updated status is displayed.
- Given a tutor cancels an accepted booking, when the status changes, then the student is notified that the lesson has been cancelled.
- Given a completed session is recorded, when the student views the booking, then its status is displayed as completed.

### INVEST Self-Check

- YES - Independent - Booking statuses can be displayed and tested without requiring the complete payment or video implementation.
- YES - Negotiable - The story does not prescribe whether statuses appear as labels, colours, icons or notifications.
- YES - Valuable - It keeps students informed about whether their lesson will take place.
- YES - Estimable - The possible booking statuses and the required outcomes are clearly listed.
- YES - Small - The story focuses on one capability: viewing and receiving updates about booking status.
- YES - Testable - Each booking state has a visible and verifiable outcome.

## Story 6: Join a Scheduled Video Session

### As a student

I want to access my scheduled online lesson
So that I can attend the one-to-one session with my tutor.

### Acceptance Criteria

- Given a student has an accepted booking, when the scheduled lesson time arrives, then the student is provided with access to the correct video room.
- Given a user is not connected to the booking, when the user attempts to access the video room, then access is denied.
- Given a booking has been cancelled, when the student attempts to join the video session, then the platform does not provide access to the room.
- Given the external video provider is unavailable, when the student attempts to join, then the platform displays an error explaining that the session cannot currently be accessed.

### INVEST Self-Check

- YES - Independent - Video access can be developed and tested using an accepted booking without depending on ratings or payouts.
- YES - Negotiable - The story does not specify whether access is provided through a button, link or embedded video interface.
- YES - Valuable - It allows the student to attend the lesson they booked.
- YES - Estimable - The valid and invalid access conditions are clearly identified.
- YES - Small - The story focuses only on providing controlled access to one scheduled video session.
- YES - Testable - Access can be verified for valid students, unauthorised users, cancelled bookings and provider failures.

## Story 7: Rate a Tutor

### As a student

I want to rate the tutor and leave a short comment after a lesson
So that I can share feedback about my experience.

### Acceptance Criteria

- Given a session has been marked as completed, when the student opens the review option, then the student can select a rating from 1 to 5 stars.
- Given the student selects a rating from 1 to 5 stars, when the review is submitted, then the rating is linked to the correct tutor and completed session.
- Given a completed session, when the student submits a one-line comment, then the comment is stored with the rating.
- Given a session has not been marked as completed, when the student attempts to submit a review, then the platform prevents the submission.

### INVEST Self-Check

- YES - Independent - The review capability can be developed and tested using a completed-session state without depending on future bookings or payouts.
- YES - Negotiable - The story does not prescribe the review-screen design or how stars and comments are visually displayed.
- YES - Valuable - It allows students to share feedback and helps future students evaluate tutors.
- YES - Estimable - The rating range, comment requirement and eligibility condition are defined.
- YES - Small - The story covers one clear capability: submitting feedback for a completed session.
- YES - Testable - The permitted rating range, comment storage and completed-session restriction can all be verified.

## Story 8: Manage Tutor Availability

### As a tutor

I want to publish, update and remove my available lesson slots
So that students can request sessions only when I am available.

### Acceptance Criteria

- Given a tutor is viewing their availability, when the tutor adds a valid one-hour time slot, then the slot becomes visible to students.
- Given an availability slot does not have a booking, when the tutor updates its date or time, then the updated details are displayed to students.
- Given an availability slot does not have a booking, when the tutor removes it, then the slot is no longer displayed to students.
- Given a slot is connected to an accepted booking, when the tutor attempts to update or remove it as ordinary availability, then the platform prevents the change.

### INVEST Self-Check

- YES - Independent - Availability management can be built and tested without requiring payment, rating or payout functionality.
- YES - Negotiable - The story does not prescribe whether tutors use a calendar, list or another interface to manage slots.
- YES - Valuable - It allows tutors to control when students can request lessons and reduces timetable conflicts.
- YES - Estimable - The add, update and remove behaviours are clearly defined.
- YES - Small - The actions belong to one cohesive capability: managing tutor availability.
- YES - Testable - Every action produces an observable change to the tutor's available lesson slots.

## Story 9: Accept or Decline a Booking Request

### As a tutor

I want to accept or decline a pending booking request
So that I can control which lessons are added to my schedule.

### Acceptance Criteria

- Given a student has submitted a booking request, when the tutor views pending requests, then the requested date and time are displayed.
- Given a booking is pending, when the tutor accepts it, then the booking status changes to accepted and the student is notified.
- Given a booking is pending, when the tutor declines it, then the booking status changes to declined and the student is notified.
- Given a booking has already been accepted or declined, when the tutor attempts to respond again, then the platform prevents another response.

### INVEST Self-Check

- YES - Independent - The decision flow can be developed and tested using pending booking requests without depending on session ratings or payouts.
- YES - Negotiable - The story does not prescribe the buttons, screen layout or notification appearance.
- YES - Valuable - It allows tutors to control their schedules and gives students a clear booking decision.
- YES - Estimable - The pending, accepted and declined states are clearly defined.
- YES - Small - Accepting and declining are two outcomes of the same booking-decision capability.
- YES - Testable - Each tutor decision results in an observable status change and student notification.

## Story 10: Cancel an Accepted Booking

### As a tutor

I want to cancel an accepted booking when sufficient notice is provided
So that the student can be informed when I cannot conduct the lesson.

### Acceptance Criteria

- Given an accepted session begins in at least 12 hours, when the tutor cancels it, then the booking status changes to cancelled.
- Given a tutor successfully cancels a booking, when the status changes, then the student is notified.
- Given an accepted session begins in less than 12 hours, when the tutor attempts to cancel it, then the platform prevents the cancellation.
- Given a booking is already cancelled or completed, when the tutor attempts to cancel it, then the platform prevents the action.

### INVEST Self-Check

- YES - Independent - Tutor cancellation can be developed and tested separately using accepted bookings with different start times.
- YES - Negotiable - The 12-hour business rule is fixed, but the cancellation interface and notification format remain negotiable.
- YES - Valuable - It informs students when a tutor cannot conduct a lesson and enforces the required notice period.
- YES - Estimable - The allowed and prohibited cancellation conditions are clearly defined.
- YES - Small - The story focuses on one action: cancelling an accepted booking.
- YES - Testable - The outcome can be verified using bookings that start before and after the 12-hour limit.

## Story 11: View Completed-Session Earnings

### As a tutor

I want to view the earnings calculation for each completed session
So that I can understand the amount payable to me after commission.

### Acceptance Criteria

- Given a session has been marked as completed, when the tutor views its earnings, then the total session fee is displayed.
- Given the total session fee is displayed, when commission is calculated, then the platform shows the 15% LearnLanka commission.
- Given the 15% commission has been deducted, when the tutor views the earnings, then the remaining 85% payable amount is displayed.
- Given a session has not been marked as completed, when the tutor views its details, then it is not included in completed-session earnings.

### INVEST Self-Check

- YES - Independent - The earnings calculation can be developed and tested separately from the weekly bank-transfer process.
- YES - Negotiable - The story does not prescribe how the fee, commission and payable amount are visually presented.
- YES - Valuable - It gives tutors a clear explanation of how their earnings are calculated.
- YES - Estimable - The total fee, 15% commission and 85% payable amount are mathematically defined.
- YES - Small - The story focuses only on calculating and displaying earnings for completed sessions.
- YES - Testable - The displayed amounts can be checked against known session fees and completion statuses.

## Story 12: View Weekly Payout Status

### As a tutor

I want to view the status of my weekly payout
So that I know whether my earnings are waiting, processing, completed or failed.

### Acceptance Criteria

- Given completed-session earnings are eligible for the next payout cycle, when the tutor views the upcoming payout, then those earnings are included in its total.
- Given a weekly payout has been submitted through Sampath Vishwa, when the tutor views it, then its current status is displayed.
- Given a payout is completed, when the tutor views the payout history, then the completed payout amount is displayed.
- Given a payout fails, when the tutor views the payout, then its status is displayed as failed.

### INVEST Self-Check

- YES - Independent - Payout status can be displayed and tested using payout records without requiring changes to tutor availability or ratings.
- YES - Negotiable - The story does not prescribe how payout history or statuses must be visually presented.
- YES - Valuable - It helps tutors understand when they should expect to receive their earnings.
- YES - Estimable - The required payout information and status outcomes are clearly identified.
- YES - Small - The story focuses on one capability: viewing weekly payout information.
- YES - Testable - Each payout state and total can be verified using known payout records.

## Story 13: Review a Tutor Application

##### As an Operations Admin

I want to review and decide on a tutor application
So that only vetted tutors become visible to students.

### Acceptance Criteria

- Given a tutor application has been submitted, when the Operations Admin opens it, then the submitted tutor information is displayed.
- Given the Operations Admin selects approve, when the decision is confirmed, then the application status changes to approved.
- Given the Operations Admin selects reject, when the decision is confirmed, then the application status changes to rejected.
- Given a tutor application has not been approved, when a student performs a tutor search, then that tutor is not displayed in the results.

### INVEST Self-Check

- YES - Independent - Application review can be developed and tested separately from bookings, payments and video sessions.
- YES - Negotiable - The story does not define the visual review layout or the detailed business contents of the vetting checklist.
- YES - Valuable - It prevents unapproved tutors from appearing to students.
- YES - Estimable - The story is limited to displaying an application, recording a decision and controlling search visibility.
- YES - Small - The story covers one administrative capability: approving or rejecting a tutor application.
- YES - Testable - Approval, rejection and tutor visibility each have observable outcomes.

## Story 14: Investigate a Failed Student Payment

##### As an Operations Admin

I want to view the details of a failed student payment
So that I can investigate the issue and support the affected student.

### Acceptance Criteria

- Given a student payment has failed, when the Operations Admin opens the transaction, then the payment status and available PayHere reference are displayed.
- Given the failed payment is linked to a booking, when the Operations Admin views the transaction, then the related booking and student information are displayed.
- Given the Operations Admin searches using a valid payment reference, when the matching transaction exists, then its failure details are displayed.
- Given the Operations Admin searches using an invalid payment reference, when no matching transaction exists, then the platform displays that no transaction was found.

### INVEST Self-Check

- YES - Independent - Failed-payment investigation can be developed and tested separately from tutor payouts and video sessions.
- YES - Negotiable - The story does not prescribe the admin-screen design or internal search method.
- YES - Valuable - It helps the Operations Admin investigate payment issues and assist affected students.
- YES - Estimable - The required transaction, booking and provider-reference details are identified.
- YES - Small - The story focuses only on investigating failed student payments.
- YES - Testable - Valid and invalid payment references produce clear observable outcomes.

## Story 15: Investigate a Failed Tutor Payout

##### As an Operations Admin

I want to view the details of a failed tutor payout
So that I can investigate the issue and support the affected tutor.

### Acceptance Criteria

- Given a tutor payout has failed, when the Operations Admin opens the payout, then the payout status and available Sampath Vishwa reference are displayed.
- Given the failed payout is connected to a tutor, when the Operations Admin views it, then the tutor and payout amount are displayed.
- Given the Operations Admin searches using a valid payout reference, when the matching payout exists, then its failure details are displayed.
- Given the Operations Admin searches using an invalid payout reference, when no matching payout exists, then the platform displays that no payout was found.

### INVEST Self-Check

- YES - Independent - Failed-payout investigation can be developed and tested separately from student payments and booking management.
- YES - Negotiable - The story does not prescribe the admin-screen design or technical connection with Sampath Vishwa.
- YES - Valuable - It helps the Operations Admin investigate why a tutor has not received expected earnings.
- YES - Estimable - The required payout, tutor and provider-reference information is clearly identified.
- YES - Small - The story focuses only on investigating failed tutor payouts.
- YES - Testable - Valid, failed and missing payout records have observable outcomes.

## Story 16: Process a Personal-Data Deletion Request

##### As an Operations Admin

I want to review and update personal-data deletion requests
So that LearnLanka can respond to users' privacy requests.

### Acceptance Criteria

- Given a student or tutor submits a deletion request, when the Operations Admin views privacy requests, then the request is displayed with a pending status.
- Given a deletion request has been reviewed, when the Operations Admin approves it, then the request status changes to approved.
- Given an approved deletion request has been completed, when the Operations Admin records completion, then the request status changes to completed.
- Given information must be retained for a valid legal reason, when the request cannot be fully completed, then the Operations Admin can record the reason and update the request status accordingly.

### INVEST Self-Check

- YES - Independent - The deletion-request workflow can be developed and tested separately from bookings, payments and tutor availability.
- YES - Negotiable - The story defines required outcomes without prescribing the admin-screen design or internal deletion process.
- YES - Valuable - It supports users' privacy rights and the platform's data-protection responsibilities.
- YES - Estimable - The request statuses and administrative actions are clearly described.
- YES - Small - The story focuses on reviewing and updating one type of privacy request.
- YES - Testable - Each administrative action produces a visible request status and recorded outcome.

## Story 17: Handle Payment After Tutor Decline or Cancellation

### As a student

I want my payment to be handled clearly when a tutor declines or cancels my booking
So that I know whether my money will be returned and when I should expect it.

### Acceptance Criteria

- Given a student has paid for a booking, when the tutor declines the booking, then the student is informed of the payment outcome and any refund status.
- Given a student has paid for an accepted booking, when the tutor cancels it with at least 12 hours' notice, then the payment is handled according to the confirmed cancellation and refund policy.
- Given a refund is initiated, when the student views the booking, then the current refund status is displayed.

### INVEST Self-Check

- YES - Independent - The story represents the payment outcome following a decline or cancellation and can be considered separately from tutor search and video-session access.
- YES - Negotiable - The exact refund process, timing and communication method remain open for stakeholder discussion.
- YES - Valuable - It protects students from losing money or being left uncertain when a tutor cannot conduct the lesson.
- NO - Estimable - The team cannot estimate the complete story until the founders confirm payment timing, refund eligibility, refund amount, processing method and expected completion time.
- YES - Small - The story focuses on one business outcome: handling an existing payment after a tutor decline or cancellation.
- NO - Testable - The trigger conditions can be tested, but the final expected result cannot be fully verified until the refund policy is defined.

Note: The founders must clarify whether payment is captured before tutor acceptance and what refund rules apply to tutor declines and cancellations.

## Story 18: Review a Reported Rating or Comment

##### As an Operations Admin

I want to review a rating or comment reported by a student or tutor
So that inappropriate or unfair feedback can be handled according to LearnLanka's moderation policy.

### Acceptance Criteria

- Given a rating or comment has been reported, when the Operations Admin opens the report, then the rating, comment, related session and reporting user are displayed.
- Given the Operations Admin has reviewed the report, when a moderation decision is recorded, then the report status is updated.
- Given a moderation action is taken, when the related user views the rating or comment, then the result reflects the permitted moderation action.

### INVEST Self-Check

- YES - Independent - Reported feedback can be reviewed separately from payment, payout and video-session features.
- YES - Negotiable - The review process and exact moderation actions can be discussed with the founders.
- YES - Valuable - It allows LearnLanka to respond to abusive, irrelevant or unfair feedback.
- NO - Estimable - The team cannot confidently estimate the complete story until the founders define report reasons, moderation actions and appeal rules.
- YES - Small - The story focuses only on reviewing one reported rating or comment.
- NO - Testable - The report workflow can be tested, but the final expected moderation result is unclear until the allowed actions and decision rules are defined.

## Story 19: Investigate a Booking or Session Dispute

##### As an Operations Admin

I want to view the history of a disputed booking and session
So that I can understand what happened and support the student and tutor.

### Acceptance Criteria

- Given a student or tutor reports a dispute, when the Operations Admin opens the related booking, then the booking status history is displayed.
- Given the booking has payment, cancellation or session records, when the Operations Admin reviews the dispute, then the available related records are displayed.
- Given the Operations Admin completes the investigation, when an outcome is recorded, then it is linked to the disputed booking.

### INVEST Self-Check

- YES - Independent - Booking and session records can be reviewed separately from tutor search and availability management.
- YES - Negotiable - The story does not prescribe the investigation screen, process or exact decision.
- YES - Valuable - It gives the Operations Admin the information needed to respond to disputes.
- NO - Estimable - The investigation effort cannot be fully estimated until the founders define the available evidence, possible outcomes and escalation process.
- YES - Small - The story focuses on investigating one disputed booking or session.
- NO - Testable - Record visibility can be tested, but the completed dispute outcome cannot be fully verified until the permitted resolutions are defined.
