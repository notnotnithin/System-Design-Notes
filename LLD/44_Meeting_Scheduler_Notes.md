# Design a Meeting Scheduler (LLD #44)

<!-- nav -->
⬅ [43 Object Pool Pattern](43_Object_Pool_Pattern_Notes.md) · [Index](00_LLD_Index.md)
<!-- nav -->

Sources: [Code on GitLab (Meeting scheduler mock interview)](https://gitlab.com/shrayansh8/lld-lowleveldesign/-/tree/main/src/main/java/com/conceptcoding/interviewquestions/meetingscheduler?ref_type=heads) (this topic has no number or PDF in your links)

## The problem
Design a system like a calendar and room-booking tool. An organiser invites people to a meeting at a given time. The system finds a **free meeting room** big enough, books it, puts the meeting in everyone's calendar and sends **invites**. People respond, and meetings can be cancelled.

## Requirements
- Users, each with a **calendar**.
- Meeting rooms with a **capacity** and their own booked time slots.
- Schedule a meeting: pick a room that fits all participants and is free during the interval.
- Notify participants of the invite (and of a cancellation).
- Participants respond: `ACCEPTED`, `DECLINED`, `TENTATIVE` or `NOT_RESPONDED`.
- Cancel a meeting: free the room and remove it from the calendars.

## Step 1 – Objects
| Object | Job |
|---|---|
| `User` | name, email, a `Calendar`; can view the calendar and respond to an invitation |
| `ParticipantType`, `ParticipantResponse` | `ORGANIZER` / `ATTENDEE`; the four response values |
| `TimeInterval` | start and end time; `overlapsWith(other)` |
| `Meeting` | id, subject, interval, organiser, room and a map of attendee to response |
| `Calendar` | a map of `TimeInterval` to `Meeting`; add and remove a meeting |
| `MeetingRoom` | id, name, capacity, booked slots, its own calendar; `isAvailableFor(interval, capacity)` |
| `Notification` | sends invite and cancellation messages |
| `MeetingScheduler` | the coordinator: owns all rooms; `scheduleMeeting`, `cancelMeeting`, room search and booking |

## Step 2 – Class diagram

**Diagram – Meeting scheduler**

```mermaid
classDiagram
  class Calendar {
    -Map~TimeInterval,Meeting~ meetings
    +addMeeting() void
    +removeMeeting() void
  }
  class Meeting {
    -String meetingId
    -String subject
    -TimeInterval timeInterval
    -User organizer
  }
  class MeetingRoom {
    -int id
    -String name
    -int capacity
    -List~TimeInterval~ bookedSlots
    +viewCalendar() void
    +addBookedSlot() void
    +removeBookedSlot() void
  }
  class MeetingScheduler {
    -List~MeetingRoom~ allMeetingRooms
    +scheduleMeeting() Meeting
    +cancelMeeting() void
    +addMeetingRoom() void
  }
  class Notification {
    -int notificationId
    -String message
    +sendMeetingInviteNotification() void
    +sendCancelMeetingNotification() void
  }
  class ParticipantResponse {
    <<enumeration>>
    ACCEPTED
    DECLINED
    TENTATIVE
    NOT_RESPONDED
  }
  class ParticipantType {
    <<enumeration>>
    ORGANIZER
    ATTENDEE
  }
  class TimeInterval {
    -LocalDateTime startTime
    -LocalDateTime endTime
    +overlapsWith() boolean
  }
  class User {
    -String name
    -String email
    -Calendar calendar
    -ParticipantType participantType
    +viewCalendar() void
    +respondToInvitation() void
  }
  Calendar --> "*" Meeting
  Calendar --> "*" TimeInterval
  Meeting --> MeetingRoom
  Meeting --> "*" ParticipantResponse
  Meeting --> TimeInterval
  Meeting --> User
  Meeting --> "*" User
  MeetingRoom --> Calendar
  MeetingRoom --> "*" TimeInterval
  MeetingScheduler --> "*" MeetingRoom
  User --> Calendar
  User --> ParticipantType
```

## Step 3 – Scheduling a meeting

```mermaid
sequenceDiagram
  participant O as Organiser
  participant S as MeetingScheduler
  participant R as MeetingRoom
  participant N as Notification
  O->>S: scheduleMeeting(organiser, participants, subject, timeInterval)
  S->>R: find a room: isAvailableFor(interval, capacity = number of participants)
  R-->>S: room (or none: stop and return null)
  S->>S: create the Meeting
  S->>S: add it to the organiser's and every participant's Calendar
  S->>R: bookSlot (add the interval to the room's booked slots and calendar)
  S->>N: send invites to all participants
  N-->>O: invites sent
```

```mermaid
sequenceDiagram
  participant U as Participant
  participant M as Meeting
  U->>M: respondToInvitation(ACCEPTED / DECLINED / TENTATIVE)
  M->>M: update attendees map: user to response
```

```mermaid
stateDiagram-v2
  [*] --> NOT_RESPONDED: invite sent
  NOT_RESPONDED --> ACCEPTED
  NOT_RESPONDED --> DECLINED
  NOT_RESPONDED --> TENTATIVE
  TENTATIVE --> ACCEPTED
  TENTATIVE --> DECLINED
```

*Find a room, create the meeting, book, notify*

```java
public Meeting scheduleMeeting(User organiser, Map<User, ParticipantResponse> participants, String subject, TimeInterval timeInterval) {
    // Check for available meeting rooms with available time slots
    MeetingRoom room = checkForAvailableRooms(participants.size(), timeInterval);
    if (room == null) {
        System.out.println("===>>> No available meeting room");
        return null;
    }
    System.out.println("\n===>>> Available meeting room found");
    // Create meeting
    Meeting meeting = new Meeting(getMeetingId(), subject, timeInterval, organiser, participants, room);

    // Add meeting to participants(both organiser and participants) calendar
    organiser.getCalendar().addMeeting(meeting);
    for (User user : participants.keySet()) {
        user.getCalendar().addMeeting(meeting);
    }
    // Add meeting to meeting room calendar
    bookSlot(room, meeting);

    // Notify participants
    notifyParticipantsWithInvites(meeting);
    simulateUserResponsesForInvites(meeting);
    System.out.println("[+] Meeting scheduled successfully!!");

    return meeting;
}
```

## Step 4 – Time overlap (the key logic)
Two intervals **overlap** unless one ends before the other starts. A room is free for a new meeting only if it has **no booked slot that overlaps** the new interval, and its capacity is at least the number of participants.

```mermaid
graph LR
  A["Existing: 10:00 to 11:00"] --> Q{"new interval overlaps?"}
  Q -->|"new 09:00 to 10:00 (touches, treated as overlap in the code)"| X["Conflict"]
  Q -->|"new 11:30 to 12:30"| OK["No overlap: room is free"]
```

*Overlap test*

```java
public boolean overlapsWith(TimeInterval newInterval) {
    // if the current interval ends before the new interval starts - valid: overlaps? false
    // if the current interval starts after the new interval ends - valid: overlaps? false
    return !this.endTime.isBefore(newInterval.startTime) &&
            !this.startTime.isAfter(newInterval.endTime);
}
```

*The code treats meetings that touch at the end point (one ends at 10:00 and the next starts at 10:00) as overlapping. Many designs use strict `<` and `>` so back-to-back meetings are allowed.*

## Design points to mention
- **Rooms and users each have a calendar**, so you can ask "what is booked?" for either one.
- **Search strategy:** the code takes the first suitable room. You can add a strategy for choosing (smallest room that fits, nearest floor, ...) ([Strategy](02_Strategy_Pattern_Notes.md)).
- **Notifications** can be made pluggable (email, Slack, push) with the [Observer pattern](03_Observer_Pattern_Notes.md).
- **Concurrency:** two organisers booking the same room at once need an atomic check-and-book (the same idea as [Car Rental](09_Car_Rental_System_Notes.md) and [BookMyShow](14_BookMyShow_Notes.md)).
- **Extensions:** recurring meetings, time zones, finding a common free time for all attendees, room equipment, waiting list for rooms.

---

<!-- nav -->
⬅ [43 Object Pool Pattern](43_Object_Pool_Pattern_Notes.md) · [Index](00_LLD_Index.md)
<!-- nav -->
