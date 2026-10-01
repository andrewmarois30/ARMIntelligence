#!/usr/bin/env python3
"""Gate one paper-agent session by local window AND an authoritative market clock."""
import datetime as dt
import json
import os
import subprocess
import sys
from zoneinfo import ZoneInfo

UTC = dt.timezone.utc

def eligibility(clock, now):
    local = now.astimezone(ZoneInfo('America/New_York'))
    if local.weekday() >= 5 or not dt.time(9,30) <= local.time().replace(tzinfo=None) < dt.time(16):
        return 0
    if clock.get('is_open') is not True:
        return 0
    stamp = dt.datetime.fromisoformat(clock['timestamp'].replace('Z','+00:00'))
    close = dt.datetime.fromisoformat(clock['next_close'].replace('Z','+00:00'))
    if stamp.tzinfo is None or close.tzinfo is None:
        raise ValueError('Clock timestamps must include timezone')
    if abs((now-stamp).total_seconds()) > 60:
        raise ValueError('Market clock is stale')
    cutoff = dt.datetime.combine(local.date(),dt.time(16),local.tzinfo)
    return max(0,int((min(close,cutoff)-now).total_seconds())-5)

def command(name):
    argv = json.loads(os.environ[name])
    if not isinstance(argv,list) or not argv or not all(isinstance(x,str) for x in argv):
        raise ValueError(name+' must be a nonempty JSON array of strings')
    return argv

def main():
    now=dt.datetime.now(UTC)
    local=now.astimezone(ZoneInfo('America/New_York'))
    if local.weekday()>=5 or not dt.time(9,30)<=local.time().replace(tzinfo=None)<dt.time(16):
        print('WAIT: outside 09:30–16:00 America/New_York',flush=True)
        return 75
    clock_run=subprocess.run(command('MARKET_CLOCK_COMMAND_JSON'),capture_output=True,text=True,timeout=20)
    if clock_run.returncode:
        raise RuntimeError('Market clock command failed; session not started')
    seconds=eligibility(json.loads(clock_run.stdout),dt.datetime.now(UTC))
    if seconds<=0:
        print('WAIT: exchange closed or closing',flush=True)
        return 75
    # Agent must independently enforce paper-only access and recheck market clock for each order.
    timeout=min(seconds,int(os.environ.get('PAPER_AGENT_MAX_SECONDS','120')))
    if timeout<=0: raise ValueError('Invalid session timeout')
    print('Starting paper agent session',flush=True)
    try:
        result=subprocess.run(command('PAPER_AGENT_COMMAND_JSON'),timeout=timeout)
        return result.returncode if result.returncode>=0 else 1
    except subprocess.TimeoutExpired:
        print('Session deadline reached; review outstanding paper orders',flush=True)
        return 1

if __name__=='__main__':
    try: sys.exit(main())
    except Exception as exc:
        # Do not echo commands, credentials, broker responses, or environment values.
        print('Wake blocked: '+type(exc).__name__+'. Check configuration and clock adapter.',file=sys.stderr)
        sys.exit(1)
