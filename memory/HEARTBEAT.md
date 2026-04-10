# Heartbeat checklist

Skills marked `HasHeartbeat` run on their own cadences. This file is the prose
checklist the LLM reads on the rare heartbeat ticks that invoke reasoning
(e.g. the daily 08:00 morning briefing).

On each invocation, consider in order:
1. System health — is anything broken?
2. Home Assistant anomalies — anything out of pattern?
3. Health trends — anything notable in the last 7 days?
4. Return `HEARTBEAT_OK` if nothing needs attention.
