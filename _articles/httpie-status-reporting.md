---
title:  "Reporting Download Progress in Httpie"
layout: default
language: Python
project: HTTPie
tags: cli,status-reporting,spinner
---

# Reporting Download Progress in Httpie

* **Status**: DRAFT
* **Project name**: HTTPie
* **Example name**: HTTPie's Download Progress Reporter
* **Project home page**: https://github.com/httpie/httpie
* **Programming language(s)**: Python
* **Frameworks, libraries used:** N/A
* **Tags:** cli,status-reporting,spinner

## Context

HTTPie (pronounced aitch-tee-tee-pie) is a command-line HTTP client. Its goal is to make CLI interaction with web services as human-friendly as possible.

## Problem

HTTPie features a download mode in which it acts similarly to wget. When enabled using the --download, -d flag, a progress bar must be shown while the response body is being saved to a file.

## Overview

Status reporting runs in its own thread. It wakes up every `tick` seconds, calculates metrics (download percentage, downloaded size, speed, eta) and writes them to console. When it's done, it writes a summary.

The speed is calculated on the interval since the last update. ETA is calculated simply as `(total_size - downloaded) / speed`.

## Implementation details

[The code in question](https://github.com/httpie/httpie/blob/64c31d554a367abf876bd355f07dca6e41476c3f/httpie/downloads.py#L369-L480) is rather short:

```python
class ProgressReporterThread(threading.Thread):
    """
    Reports download progress based on its status.
    Uses threading to periodically update the status (speed, ETA, etc.).
    """

    def __init__(
        self,
        status: DownloadStatus,
        output: IO,
        tick=.1,
        update_interval=1
    ):
        super().__init__()
        self.status = status
        self.output = output
        self._tick = tick
        self._update_interval = update_interval
        self._spinner_pos = 0
        self._status_line = ''
        self._prev_bytes = 0
        self._prev_time = time()
        self._should_stop = threading.Event()

    def stop(self):
        """Stop reporting on next tick."""
        self._should_stop.set()

    def run(self):
        while not self._should_stop.is_set():
            if self.status.has_finished:
                self.sum_up()
                break

            self.report_speed()
            sleep(self._tick)

    def report_speed(self):

        now = time()

        if now - self._prev_time >= self._update_interval:
            downloaded = self.status.downloaded
            try:
                speed = ((downloaded - self._prev_bytes)
                         / (now - self._prev_time))
            except ZeroDivisionError:
                speed = 0

            if not self.status.total_size:
                self._status_line = PROGRESS_NO_CONTENT_LENGTH.format(
                    downloaded=humanize_bytes(downloaded),
                    speed=humanize_bytes(speed),
                )
            else:
                try:
                    percentage = downloaded / self.status.total_size * 100
                except ZeroDivisionError:
                    percentage = 0

                if not speed:
                    eta = '-:--:--'
                else:
                    s = int((self.status.total_size - downloaded) / speed)
                    h, s = divmod(s, 60 * 60)
                    m, s = divmod(s, 60)
                    eta = f'{h}:{m:0>2}:{s:0>2}'

                self._status_line = PROGRESS.format(
                    percentage=percentage,
                    downloaded=humanize_bytes(downloaded),
                    speed=humanize_bytes(speed),
                    eta=eta,
                )

            self._prev_time = now
            self._prev_bytes = downloaded

        self.output.write(
            f'{CLEAR_LINE} {SPINNER[self._spinner_pos]} {self._status_line}'
        )
        self.output.flush()

        self._spinner_pos = (self._spinner_pos + 1
                             if self._spinner_pos + 1 != len(SPINNER)
                             else 0)

    def sum_up(self):
        actually_downloaded = (
            self.status.downloaded - self.status.resumed_from)
        time_taken = self.status.time_finished - self.status.time_started

        self.output.write(CLEAR_LINE)

        try:
            speed = actually_downloaded / time_taken
        except ZeroDivisionError:
            # Either time is 0 (not all systems provide `time.time`
            # with a better precision than 1 second), and/or nothing
            # has been downloaded.
            speed = actually_downloaded

        self.output.write(SUMMARY.format(
            downloaded=humanize_bytes(actually_downloaded),
            total=(self.status.total_size
                   and humanize_bytes(self.status.total_size)),
            speed=humanize_bytes(speed),
            time=time_taken,
        ))
        self.output.flush()
```

[Format strings](https://github.com/httpie/httpie/blob/64c31d554a367abf876bd355f07dca6e41476c3f/httpie/downloads.py#L25-L34) are defined above:
```python
CLEAR_LINE = '\r\033[K'
PROGRESS = (
    '{percentage: 6.2f} %'
    ' {downloaded: >10}'
    ' {speed: >10}/s'
    ' {eta: >8} ETA'
)
PROGRESS_NO_CONTENT_LENGTH = '{downloaded: >10} {speed: >10}/s'
SUMMARY = 'Done. {downloaded} in {time:0.5f}s ({speed}/s)\n'
SPINNER = '|/-\\'
```

## Testing

Automated testing for this functionality seems to be lacking.

## Observations

* The spinner simply iterates between the 4 states: vertical line, forward slash, horizontal line, back slash.
* Updating spinner position could be simplified (arguably it would make it less clear, though):
```python
self._spinner_pos = (self._spinner_pos + 1
                        if self._spinner_pos + 1 != len(SPINNER)
                        else 0)
```
to
```python
self._spinner_pos = (self._spinner_pos + 1) % len(SPINNER)
```
* To clear the line, it prints [this magic string](https://github.com/httpie/httpie/blob/64c31d554a367abf876bd355f07dca6e41476c3f/httpie/downloads.py#L25): ```CLEAR_LINE = '\r\033[K'```. It's a [CSI sequence](https://en.wikipedia.org/wiki/ANSI_escape_code#CSI_(Control_Sequence_Introducer)_sequences).

## Related

N/A

## References

* [Github repo](https://github.com/httpie/httpie)
