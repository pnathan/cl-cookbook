---
title: Dates and Times
---

Common Lisp provides two different ways of looking at time: universal time,
meaning time in the "real world", and run time, meaning time as seen by your
computer's CPU. We will deal with both of them separately.

<a name="univ"></a>

## Universal Time

Universal time is represented as the number of seconds that have elapsed since
00:00 of January 1, 1900 in the GMT time zone. The function
[`GET-UNIVERSAL-TIME`](http://www.lispworks.com/documentation/HyperSpec/Body/f_get_un.htm)
returns the current universal time:

~~~lisp
* (get-universal-time)
3220993326
~~~

Of course this value is not very readable, so you can use the function
[`DECODE-UNIVERSAL-TIME`](http://www.lispworks.com/documentation/HyperSpec/Body/f_dec_un.htm)
to turn it into a "calendar time" representation:

~~~lisp
* (decode-universal-time 3220993326)
6
22
19
25
1
2002
4
NIL
5
~~~

This call returns nine values: seconds, minutes, hours, day, month, year, day of
the week, daylight savings time flag and time zone. Note that the day of the
week is represented as an integer in the range 0..6 with 0 being Monday and 6
being Sunday. Also, the time zone is represented as the number of hours you need
to add to the current time in order to get GMT time. So in this example the
decoded time would be 19:22:06 of Friday, January 25, 2002, in the EST time
zone, with no daylight savings in effect. This, of course, relies on the
computer's own clock, so make sure that it is set correctly (including the time
zone you are in and the DST flag). As a shortcut, you can use
[`GET-DECODED-TIME`](http://www.lispworks.com/documentation/HyperSpec/Body/f_get_un.htm)
to get the calendar time representation of the current time directly:

~~~lisp
* (get-decoded-time)
~~~

is equivalent to

~~~lisp
* (decode-universal-time (get-universal-time))
~~~

Here is an example of how to use these functions in a program:

~~~lisp
* (defconstant *day-names*
    '("Monday" "Tuesday" "Wednesday"
      "Thursday" "Friday" "Saturday"
      "Sunday"))
*DAY-NAMES*

* (multiple-value-bind
	(second minute hour date month year day-of-week dst-p tz)
	(get-decoded-time)
    (format t "It is now ~2,'0d:~2,'0d:~2,'0d of ~a, ~d/~2,'0d/~d (GMT~@d)"
	      hour
	      minute
	      second
	      (nth day-of-week *day-names*)
	      month
	      date
	      year
	      (- tz)))
It is now 17:07:17 of Saturday, 1/26/2002 (GMT-5)
~~~

Of course the call to `GET-DECODED-TIME` above could be replaced by
`(DECODE-UNIVERSAL-TIME n)`, where n is any integer number, to print an
arbitrary date. You can also go the other way around: the function
[`ENCODE-UNIVERSAL-TIME`](http://www.lispworks.com/documentation/HyperSpec/Body/f_encode.htm)
lets you encode a calendar time into the corresponding universal time. This
function takes six mandatory arguments (seconds, minutes, hours, date, month and
year) and one optional argument (the time zone) and it returns a universal time:

~~~lisp
* (encode-universal-time 6 22 19 25 1 2002)
3220993326
~~~

Note that the result is automatically adjusted for daylight savings time if the time zone is not supplied. If it is supplied, than Lisp assumes that the specified time zone already accounts for daylight savings time, and no adjustment is performed.

Since universal times are simply numbers, they are easier and safer to manipulate than calendar times. Dates and times should always be stored as universal times if possibile, and only converted to string representations for output purposes. For example, it is straightforward to know which of two dates came before the other, by simply comparing the two corresponding universal times with <. Another typical problem is how to compute the "temporal distance" between two given dates. Let's see how to do this with an example: specifically, we will calculate the temporal distance between the first landing on the moon (4:17pm EDT, July 20 1969) and the last takeoff of the space shuttle Challenger (11:38 a.m. EST, January 28, 1986).

~~~lisp
* (setq *moon* (encode-universal-time 0 17 16 20 7 1969 4))
2194805820

* (setq *takeoff* (encode-universal-time 0 38 11 28 1 1986 5))
2716303080

* (- *takeoff* *moon*)
521497260
~~~

That's a bit over 52 million seconds, corresponding to 6035 days, 20 hours and 21 minutes (you can verify this by dividing that number by 60, 60 and 24 in succession). Going beyond days is a bit harder because months and years don't have a fixed length, but the above is approximately 16 and a half years.

You can in theory use differences between universal times to measure how long the execution of a part of your program took, but the universal times are represented in seconds, and this resolution will usually be too low to be useful. We will see a better method of doing this in the section about internal time.

To sum up, we have seen how to turn a universal time into a calendar time and vice-versa, how to perform calculations on universal times, and how to format calendar times in a human-readable way. The last piece that is missing is how to parse a string represented as a human-readable string (e.g. "03/11/1997") and turn it into a calendar time. Unfortunately this turns out to be very difficult in the general case, due to the multitude of different ways of writing dates and times that we use. In some cases it might not even be possible without context information: the above example can be correctly parsed both as March 11th or as November 3rd according to where you are living. In conclusion, either force your users to write dates in a fixed format, or be prepared to write a very intelligent parsing function!<a name="intern"></a>

### Internal Time

Internal time is the time as measured by your Lisp environment, using your computer's clock. It differs from universal time in three important respects. First, internal time is not measured starting from a specified point in time: it could be measured from the instant you started your Lisp, from the instant you booted your machine, or from any other arbitrary time point in the past. As we will see shortly, the absolute value of an internal time is almost always meaningless; only differences between internal times are useful. The second difference is that internal time is not measured in seconds, but in a (usually smaller) unit whose value can be deduced from [`INTERNAL-TIME-UNITS-PER-SECOND`](http://www.lispworks.com/documentation/HyperSpec/Body/v_intern.htm):

~~~lisp
* internal-time-units-per-second
1000
~~~

This means that in the Lisp environment used in this example, internal time is measured in milliseconds. Finally, what is being measured by the "internal time" clock? There are actually two different internal time clocks in your Lisp: one of them meaures the passage of "real" time (the same time that universal time measures, but in different units), and the other one measures the passage of CPU time, that is, the time your CPU spends doing actual computation for the current Lisp process. On most modern computers these two times will be different, since your CPU will never be entirely dedicated to your program (even on single-user machines, the CPU has to devote part of its time to processing interrupts, performing I/O, etc). The two functions used to retrieve internal times are called [`GET-INTERNAL-REAL-TIME`](http://www.lispworks.com/documentation/HyperSpec/Body/f_get_in.htm) and [`GET-INTERNAL-RUN-TIME`](http://www.lispworks.com/documentation/HyperSpec/Body/f_get__1.htm) respectively. Using them, we can solve the above problem about measuring a function's run time:

~~~lisp
* (let ((real1 (get-internal-real-time))
        (run1 (get-internal-run-time)))
    (... your call here ...)
    (let ((run2 (get-internal-run-time))
	    (real2 (get-internal-real-time)))
	(format t "Computation took:~%")
	(format t "  ~f seconds of real time~%"
		(/ (- real2 real1) internal-time-units-per-second))
	(format t "  ~f seconds of run time~%"
		(/ (- run2 run1) internal-time-units-per-second))))
~~~

A good way to see the difference between real time and run time is to test the above code using a call such as `(SLEEP 3)`. The [`SLEEP`](http://www.lispworks.com/documentation/HyperSpec/Body/f_sleep.htm) function suspends the execution of your code for the specified number of seconds. You should therefore see a real time very close to the argument of `SLEEP` and a run time very close to zero. Let's turn the above code into a macro in order to make it more general:

~~~lisp
* (defmacro timing (&body forms)
    (let ((real1 (gensym))
	    (real2 (gensym))
	    (run1 (gensym))
	    (run2 (gensym))
	    (result (gensym)))
    `(let* ((,real1 (get-internal-real-time))
	      (,run1 (get-internal-run-time))
	      (,result (progn ,@forms))
	      (,run2 (get-internal-run-time))
	      (,real2 (get-internal-real-time)))
	 (format *debug-io* ";;; Computation took:~%")
	 (format *debug-io* ";;;  ~f seconds of real time~%"
		 (/ (- ,real2 ,real1) internal-time-units-per-second))
	 (format t ";;;  ~f seconds of run time~%"
		 (/ (- ,run2 ,run1) internal-time-units-per-second))
	 ,result)))
TIMING

* (timing (sleep 1))
;;; Computation took: 0.994 seconds of real time 0.0 seconds of run
;;; time
NIL
~~~

The built-in macro [`TIME`](http://www.lispworks.com/documentation/HyperSpec/Body/m_time.htm) does roughly the same as the above macro (it executes a form and prints timing information at the end), but it also usually provides information about memory usage, time spent in garbage collection, page faults, etc. The format of the output is implementation-dependent, but in general it's pretty useful and informative. This is an example under Allegro Common Lisp 6.0: we generate a list of 100 real numbers and we measure the time it takes to sort them in ascending order.

~~~lisp
* (let ((numbers (loop for i from 1 to 100 collect (random 1.0))))
    (time (sort numbers #'<)))
; cpu time (non-gc) 0 msec user, 10 msec system
; cpu time (gc)     0 msec user, 0 msec system
; cpu time (total)  0 msec user, 10 msec system
; real time  9 msec
; space allocation:
;  3,586 cons cells, 11,704 other bytes, 0 static bytes
~~~

<a name="weekday"></a>

### Computing the day of the week

In the section about [Universal Time](#univ) we've learned enough to write a small function that computes the day of the week. Unfortunately, by definition, this function won't work for dates before January 1, 1900.

~~~lisp
* (defun day-of-week (day month year)
    "Returns the day of the week as an integer.
Monday is 0."
    (nth-value
     6
     (decode-universal-time
      (encode-universal-time 0 0 0 day month year 0)
      0)))
DAY-OF-WEEK
* (day-of-week 23 12 1965)
3
* (day-of-week 1 1 1900)
0
* (day-of-week 31 12 1899)

Type-error in KERNEL::OBJECT-NOT-TYPE-ERROR-HANDLER:
   1899 is not of type (OR (MOD 100) (INTEGER 1900))
~~~

If this is a problem for you, here's a small function by Gerald Doussot (adapted from the comp.lang.c FAQ) that will help you:

~~~lisp
(defun day-of-week (day month year)
  "Returns the day of the week as an integer.
Sunday is 0. Works for years after 1752."
  (let ((offset '(0 3 2 5 0 3 5 1 4 6 2 4)))
    (when (< month 3)
      (decf year 1))
    (mod
     (truncate (+ year
                  (/ year 4)
                  (/ (- year)
                     100)
                  (/ year 400)
                  (nth (1- month) offset)
                  day
                  -1))
     7)))
~~~
