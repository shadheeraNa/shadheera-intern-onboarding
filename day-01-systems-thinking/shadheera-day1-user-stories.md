# LearnLanka - User Story Set

## Story 1: Search Tutors by Subject, Grade, Language and Price Band

### As a student

I want to search for tutors by subject, grade, teaching language and price band

So that I can find tutors who match my learning needs and budget.

### Acceptance Criteria

* Given a student is on the tutor search page, when the student selects a subject and performs a search, then only tutors who teach that subject are displayed.
* Given the student selects a grade, teaching language and price band, when the search is performed, then every displayed tutor matches all selected filters.
* Given matching tutors are found, when the results are displayed, then each result shows the tutor's name, subjects, supported grades, teaching languages, hourly price, rating and available lesson times.
* Given no tutor matches the selected filters, when the search is performed, then the platform displays a message explaining that no matching tutors were found.

### INVEST Self-Check

YES - Independent - The tutor-search capability can be developed and tested without depending on booking, payment or video-session features.

YES - Negotiable - The story does not prescribe the search-screen layout, filter controls or technical implementation.

YES - Valuable - It allows students to find a useful shortlist of tutors who match their learning needs and budget.

YES - Estimable - The supported filters, displayed information and expected outcomes are clearly defined.

YES - Small - The story covers one cohesive capability: filtering and displaying matching tutors.

YES - Testable - Every acceptance criterion has an observable pass-or-fail outcome.

## Story 2: Fast Tutor Search Response Time

### As a student

I want tutor search results to load quickly

So that I can compare tutors without unnecessary delays.

### Acceptance Criteria

* Given valid tutor searches are performed through a Sri Lankan ISP, when response times are measured, then at least 95% of search responses are returned in less than 800 milliseconds.
* Given searches using subject, grade, teaching language and price-band filters are tested under the agreed initial operating conditions, when the 95th-percentile response time is calculated, then it remains below 800 milliseconds.

### INVEST Self-Check

YES - Independent - Search performance can be measured separately from booking, payment and video-session capabilities.

YES - Negotiable - The required outcome is defined without prescribing caching, database or infrastructure techniques.

YES - Valuable - Quick search results reduce delays while students are trying to find and compare tutors.

YES - Estimable - The response-time target, percentile and network context are clearly specified.

YES - Small - The story focuses on one non-functional concern: tutor-search response time.

YES - Testable - The response-time target is numeric and can be verified through repeatable performance testing.

## Story 3: Request an Available Lesson Slot

### As a student

I want to request an available one-hour lesson slot

So that I can reserve a suitable time with a tutor.

### Acceptance Criteria

* Given a student is viewing a tutor's available lesson times, when the student selects an available slot, then the platform allows the student to submit a booking request.
* Given the booking request is successfully submitted, when the student views the booking, then its status is displayed as pending.
* Given a booking request is pending for a time slot, when another student views the same tutor's availability, then that slot is not displayed as available.
* Given the selected slot is no longer available, when the student attempts to submit the booking request, then the platform prevents the request and asks the student to select another slot.

### INVEST Self-Check

YES - Independent - The booking-request capability can be developed and tested using existing tutor availability without depending on payment or video-session features.

YES - Negotiable - The story does not specify how the available slots or booking form must be displayed.

YES - Valuable - It allows students to request a suitable lesson time and reduces conflicting requests for the same slot.

YES - Estimable - The booking status and expected availability behaviour are clearly described.

YES - Small - The story focuses on one capability: requesting and temporarily reserving an available slot.

YES - Testable - Each criterion produces a visible booking or availability outcome.

Note: Temporarily reserving a pending slot is based on an assumption in the requirements document and must be confirmed with the founders.

## Story 4: Pay for a Booking

### As a student

I want to pay for a booking using a card or eZ Cash

So that I can complete the required payment for my lesson.

### Acceptance Criteria

* Given a booking is ready for payment, when the student selects card or eZ Cash, then the payment is processed through PayHere.
* Given PayHere confirms that the payment was successful, when LearnLanka receives the result, then the booking records the payment as successful.
* Given PayHere reports that the payment failed or was cancelled, when LearnLanka receives the result, then the booking is not marked as paid and the student is informed.
* Given the student enters card or eZ Cash payment details, when the payment is processed, then those payment credentials are not stored on LearnLanka servers.

### INVEST Self-Check

YES - Independent - The payment flow can be developed and tested using a booking that has already reached the payment stage.

YES - Negotiable - The story defines the supported payment methods and outcomes without prescribing the payment-screen design or integration structure.

YES - Valuable - It enables students to pay securely using one of the supported payment methods.

YES - Estimable - PayHere, the supported methods and the expected success and failure outcomes are identified.

YES - Small - The story focuses only on processing and recording one booking payment.

YES - Testable - Successful, failed and cancelled results can be verified, including the requirement not to store payment credentials.

## Story 5: Handle Payment After Tutor Decline or Cancellation

### As a student

I want my payment to be handled clearly when a tutor declines or cancels my booking

So that I know whether my money will be returned and when I should expect it.

### Acceptance Criteria

* Given a student has paid for a pending booking, when the tutor declines the booking, then the student is informed of the payment outcome and any refund status.
* Given a student has paid for an accepted booking, when the tutor cancels it, then the payment is handled according to the confirmed cancellation and refund policy.
* Given a refund is initiated, when the student views the related booking, then the current refund status is displayed.
* Given a refund is completed or fails, when LearnLanka receives the result, then the student is informed of the outcome.

### INVEST Self-Check

YES - Independent - The story focuses specifically on the payment outcome after a tutor decline or cancellation.

YES - Negotiable - The exact refund process, amount, timing and communication method remain open for stakeholder discussion.

YES - Valuable - It protects students from losing money or being left uncertain when a tutor cannot conduct the lesson.

NO - Estimable - The team cannot estimate the complete story until the founders confirm when payment is captured, refund eligibility, refund amount, processing method and expected completion time.

YES - Small - The story focuses on one business outcome: handling an existing payment after a tutor decline or cancellation.

NO - Testable - The refund triggers can be tested, but the final expected result cannot be fully verified until the refund policy is defined.

Note: The payment and refund rules must be clarified before this story is ready for development.

## Story 6: Manage Tutor Availability

### As a tutor

I want to publish, update and remove my available lesson slots

So that students can request sessions only when I am available.

### Acceptance Criteria

* Given a tutor is viewing their availability, when the tutor adds a valid one-hour time slot, then the slot becomes visible to students.
* Given an availability slot does not have a booking, when the tutor updates its date or time, then the updated details are displayed to students.
* Given an availability slot does not have a booking, when the tutor removes it, then the slot is no longer displayed to students.
* Given a slot is connected to an accepted booking, when the tutor attempts to update or remove it as ordinary availability, then the platform prevents the change.

### INVEST Self-Check

YES - Independent - Availability management can be developed and tested without requiring payment, rating or payout functionality.

YES - Negotiable - The story does not prescribe whether tutors use a calendar, list or another type of interface.

YES - Valuable - It allows tutors to control when students can request lessons and reduces timetable conflicts.

YES - Estimable - The add, update and remove behaviours are clearly defined.

YES - Small - The actions belong to one cohesive capability: managing tutor availability.

YES - Testable - Every action produces an observable change to the tutor's available lesson slots.

## Story 7: Accept or Decline a Booking Request

### As a tutor

I want to accept or decline a pending booking request

So that I can control which lessons are added to my schedule.

### Acceptance Criteria

* Given a student has submitted a booking request, when the tutor views pending requests, then the requested date and time are displayed.
* Given a booking is pending, when the tutor accepts it, then the booking status changes to accepted and the student is notified.
* Given a booking is pending, when the tutor declines it, then the booking status changes to declined and the student is notified.
* Given a booking has already been accepted or declined, when the tutor attempts to accept or decline the same booking again, then the platform prevents another booking decision.

### INVEST Self-Check

YES - Independent - The decision flow can be developed and tested using pending booking requests without depending on ratings or payouts.

YES - Negotiable - The story does not prescribe the buttons, screen layout or notification appearance.

YES - Valuable - It allows tutors to control their schedules and gives students a clear booking decision.

YES - Estimable - The pending, accepted and declined states are clearly defined.

YES - Small - Accepting and declining are two possible outcomes of the same booking-decision capability.

YES - Testable - Each tutor decision results in an observable status change and student notification.

Note: The payment outcome after a tutor declines a paid booking is covered separately in Story 5.

## Story 8: Cancel an Accepted Booking

### As a tutor

I want to cancel an accepted booking when sufficient notice is provided

So that the student can be informed when I cannot conduct the lesson.

### Acceptance Criteria

* Given an accepted session begins in at least 12 hours, when the tutor cancels it, then the booking status changes to cancelled.
* Given the tutor successfully cancels a booking, when the booking status changes, then the student is notified.
* Given an accepted session begins in less than 12 hours, when the tutor attempts to cancel it, then the platform prevents the cancellation.
* Given a booking is already cancelled or completed, when the tutor attempts to cancel it, then the platform prevents the action.

### INVEST Self-Check

YES - Independent - Tutor cancellation can be developed and tested using accepted bookings with different scheduled start times.

YES - Negotiable - The 12-hour business rule is fixed, but the cancellation interface and notification format remain negotiable.

YES - Valuable - It informs students when a tutor cannot conduct a lesson and enforces the required notice period.

YES - Estimable - The permitted and prohibited cancellation conditions are clearly defined.

YES - Small - The story focuses on one action: cancelling an accepted booking.

YES - Testable - The outcome can be verified using bookings that begin before and after the 12-hour limit.

Note: The payment outcome after a tutor cancels a paid booking is covered separately in Story 5.

## Story 9: Review a Tutor Application

##### As an Operations Admin

I want to review and decide on a tutor application

So that only vetted tutors become visible to students.

### Acceptance Criteria

* Given a tutor application has been submitted, when the Operations Admin opens it, then the submitted tutor information is displayed.
* Given the Operations Admin approves an application, when the decision is confirmed, then its status changes to approved and the tutor becomes eligible to appear in search results.
* Given the Operations Admin rejects an application, when the decision is confirmed, then its status changes to rejected and the tutor does not appear in search results.
* Given an application remains pending, when a student performs a tutor search, then that tutor is not displayed.

### INVEST Self-Check

YES - Independent - Application review can be developed and tested separately from bookings, payments and video sessions.

YES - Negotiable - The story does not prescribe the review-screen layout or the detailed contents of the vetting checklist.

YES - Valuable - It prevents unapproved tutors from becoming visible to students.

YES - Estimable - The story is limited to displaying an application, recording a decision and controlling search visibility.

YES - Small - The story covers one administrative capability: approving or rejecting a tutor application.

YES - Testable - Pending, approved and rejected applications each produce observable outcomes.

Note: The exact qualifications and evidence used for vetting must still be confirmed with the founders.

## Story 10: Investigate a Booking or Session Dispute

##### As an Operations Admin

I want to view the history and available records of a disputed booking or session

So that I can understand what happened and support the student and tutor.

### Acceptance Criteria

* Given a student or tutor reports a dispute with a valid booking reference, when the Operations Admin opens the booking, then its status history is displayed.
* Given payment, cancellation or session records exist for the disputed booking, when the Operations Admin reviews it, then the available records and timestamps are displayed.
* Given the Operations Admin records an investigation note, when it is saved, then the note is linked to the disputed booking.
* Given the Operations Admin searches using an invalid booking reference, when no matching booking exists, then the platform displays that no booking was found.

### INVEST Self-Check

YES - Independent - Dispute information can be viewed separately from tutor search, availability management and new booking creation.

YES - Negotiable - The story does not prescribe the investigation-screen layout or the final dispute-resolution policy.

YES - Valuable - It gives the Operations Admin the information needed to understand and investigate reported problems.

YES - Estimable - The required booking history, related records and investigation-note capability are clearly identified.

YES - Small - The story focuses on investigating one disputed booking or session rather than resolving every possible type of dispute.

YES - Testable - Valid and invalid booking references, record visibility and saved investigation notes all have observable outcomes.

Note: The possible dispute resolutions, refund decisions and escalation process must be defined separately before a complete dispute-resolution story can be written.
