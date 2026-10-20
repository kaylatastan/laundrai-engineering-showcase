# Media

## Product Video

[Watch the LaundrAI product introduction video on YouTube](https://www.youtube.com/watch?v=4BU2yrt2p6U)

A short product introduction video for LaundrAI. For the running prototype, see the screenshots below.

## Screenshots

Captured from the running prototype in a desktop browser in September 2026; later UI refinements may not be reflected. Screenshots contain fictional demo names and sample interactions created for demonstration purposes. No real user information is shown.

### Dashboard with the assistant panel

![Dashboard with the LaundrAI assistant panel](screenshots/dashboard-assistant.png)

The dashboard with a machine detail panel open and the LaundrAI assistant answering availability questions. All ten machines were in use when this was captured, so the assistant reports that none are available. The **"Weekly usage" chart shows static placeholder values**, not measured data.

### Machines

![Machines screen with status filters](screenshots/machines.png)

Live machine list streamed from Firestore, with search and status filters (All, Available, Running, Reserved, Unavailable) and per-status counts.

### Schedule

![Schedule screen](screenshots/schedule.png)

A day view of live machine status with a per-machine filter. Reservations start immediately; booking future time slots is not implemented.

### Sign in

![Sign-in screen](screenshots/sign-in.png)

Email/password sign-in backed by Firebase Authentication.
