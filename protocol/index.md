---
layout: default
title: 'Protocol'
---

# Gearman Protocol

This file is maintained in the [gearmand project on GitHub](https://github.com/gearman/gearmand/blob/master/PROTOCOL),
so any modifications should be made there.

{% highlight text %}
Gearman Protocol
----------------

The Gearman protocol operates over TCP, port 4730 by default. It
previously operated on port 7003, but this conflicted with the AFS
port range and the new port (4730) was assigned by IANA. Communication
happens between either a client and job server, or between a worker
and job server. In either case, the protocol consists of packets
containing requests and responses. All packets sent to a job server
are considered requests, and all packets sent from a job server are
considered responses. A simple configuration may look like:

----------     ----------     ----------     ----------
| Client |     | Client |     | Client |     | Client |
----------     ----------     ----------     ----------
     \             /              \             /
      \           /                \           /
      --------------               --------------
      | Job Server |               | Job Server |
      --------------               --------------
            |                            |
    ----------------------------------------------
    |              |              |              |
----------     ----------     ----------     ----------
| Worker |     | Worker |     | Worker |     | Worker |
----------     ----------     ----------     ----------

Initially, the workers register functions they can perform with each
job server. Clients will then connect to a job server and issue a
request to a job to be run. The job server then notifies each worker
that can perform that job (based on the function it registered) that
a new job is ready. The first worker to wake up and retrieve the job
will then execute it.

All communication between workers or clients and the job server
are binary. There is also a line-based text protocol used by
administrative clients. This part of the protocol is text based so a
custom administrative utility is not required (instead, 'telnet' or
'nc' can be used). This is documented under "Administrative Protocol".


Binary Packet
-------------

Requests and responses are encapsulated by a binary packet. A binary
packet consists of a header which is optionally followed by data. The
header is:

4 byte magic code - This is either "\0REQ" for requests or "\0RES"
                    for responses.

4 byte type       - A big-endian (network-order) integer containing
                    an enumerated packet type. Possible values are:

                    #   Name                Magic  Type
                    1   CAN_DO              REQ    Worker
                    2   CANT_DO             REQ    Worker
                    3   RESET_ABILITIES     REQ    Worker
                    4   PRE_SLEEP           REQ    Worker
                    5   (unused)            -      -
                    6   NOOP                RES    Worker
                    7   SUBMIT_JOB          REQ    Client
                    8   JOB_CREATED         RES    Client
                    9   GRAB_JOB            REQ    Worker
                    10  NO_JOB              RES    Worker
                    11  JOB_ASSIGN          RES    Worker
                    12  WORK_STATUS         REQ    Worker
                                            RES    Client
                    13  WORK_COMPLETE       REQ    Worker
                                            RES    Client
                    14  WORK_FAIL           REQ    Worker
                                            RES    Client
                    15  GET_STATUS          REQ    Client
                    16  ECHO_REQ            REQ    Client/Worker
                    17  ECHO_RES            RES    Client/Worker
                    18  SUBMIT_JOB_BG       REQ    Client
                    19  ERROR               RES    Client/Worker
                    20  STATUS_RES          RES    Client
                    21  SUBMIT_JOB_HIGH     REQ    Client
                    22  SET_CLIENT_ID       REQ    Worker
                    23  CAN_DO_TIMEOUT      REQ    Worker
                    24  ALL_YOURS           REQ    Worker
                    25  WORK_EXCEPTION      REQ    Worker
                                            RES    Client
                    26  OPTION_REQ          REQ    Client/Worker
                    27  OPTION_RES          RES    Client/Worker
                    28  WORK_DATA           REQ    Worker
                                            RES    Client
                    29  WORK_WARNING        REQ    Worker
                                            RES    Client
                    30  GRAB_JOB_UNIQ       REQ    Worker
                    31  JOB_ASSIGN_UNIQ     RES    Worker
                    32  SUBMIT_JOB_HIGH_BG  REQ    Client
                    33  SUBMIT_JOB_LOW      REQ    Client
                    34  SUBMIT_JOB_LOW_BG   REQ    Client
                    35  SUBMIT_JOB_SCHED    REQ    Client
                    36  SUBMIT_JOB_EPOCH    REQ    Client
                    37  SUBMIT_REDUCE_JOB   REQ    Client
                    38  SUBMIT_REDUCE_JOB_BACKGROUND REQ    Client
                    39  GRAB_JOB_ALL        REQ    Worker
                    40  JOB_ASSIGN_ALL      RES    Worker
                    41  GET_STATUS_UNIQUE   REQ    Client
                    42  STATUS_RES_UNIQUE   RES    Client


4 byte size       - A big-endian (network-order) integer containing
                    the size of the data being sent after the header.

Arguments given in the data part are separated by a NULL byte, and
the last argument is determined by the size of data after the last
NULL byte separator. All job handle arguments must not be longer than
64 bytes, including NULL terminator.


Client/Worker Requests
----------------------

These request types may be sent by either a client or a worker:

ECHO_REQ

    When a job server receives this request, it simply generates a
    ECHO_RES packet with the data. This is primarily used for testing
    or debugging.

    Arguments:
    - Opaque data that is echoed back in response.


Client/Worker Responses
-----------------------

These response types may be sent to either a client or a worker:

ECHO_RES

    This is sent in response to a ECHO_REQ request. The server doesn't
    look at or modify the data argument, it just sends it back.

    Arguments:
    - Opaque data that is echoed back in response.

ERROR

    This is sent whenever the server encounters an error and needs
    to notify a client or worker.

    Arguments:
    - NULL byte terminated error code string.
    - Error text.


Client Requests
---------------

These request types may only be sent by a client:

SUBMIT_JOB, SUBMIT_JOB_BG,
SUBMIT_JOB_HIGH, SUBMIT_JOB_HIGH_BG,
SUBMIT_JOB_LOW, SUBMIT_JOB_LOW_BG

    A client issues one of these when a job needs to be run. The
    server will then assign a job handle and respond with a JOB_CREATED
    packet.

    If one of the BG versions is used, the client is not updated with
    status or notified when the job has completed (it is detached).

    The Gearman job server queue is implemented with three levels:
    normal, high, and low. Jobs submitted with one of the HIGH versions
    always take precedence, and jobs submitted with the normal versions
    take precedence over the LOW versions.

    The unique ID can be used by the server to reduce queue length. If a
    job with the same Unique ID has already been submitted, the server
    may attach this request to the already existing job. This includes
    jobs already in progress, in which case non-background jobs will be
    sent the same result as background jobs. This is known commonly as
    "coalescing".

    Arguments:
    - NULL byte terminated function name.
    - NULL byte terminated unique ID.
    - Opaque data that is given to the function as an argument.

SUBMIT_REDUCE_JOB, SUBMIT_REDUCE_JOB_BACKGROUND

    Works like the other SUBMIT_JOB commands, but adds a reducer argument.

    Arguments:
    - NULL byte terminated function name.
    - NULL byte terminated unique ID.
    - NULL byte terminated reducer.
    - Opaque data that is given to the function as an argument.

SUBMIT_JOB_SCHED

    Just like SUBMIT_JOB_BG, but run job at given time instead of
    immediately. This is not currently used and may be removed.

    Arguments:
    - NULL byte terminated function name.
    - NULL byte terminated unique ID.
    - NULL byte terminated minute (0-59).
    - NULL byte terminated hour (0-23).
    - NULL byte terminated day of month (1-31).
    - NULL byte terminated month (1-12).
    - NULL byte terminated day of week (0-6, 0 = Monday).
    - Opaque data that is given to the function as an argument.

SUBMIT_JOB_EPOCH

    Just like SUBMIT_JOB_BG, but run job at given time instead of
    immediately. This is not currently used and may be removed.

    Arguments:
    - NULL byte terminated function name.
    - NULL byte terminated unique ID.
    - NULL byte terminated epoch time.
    - Opaque data that is given to the function as an argument.

GET_STATUS

    A client issues this to get status information for a submitted job.

    Arguments:
    - Job handle that was given in JOB_CREATED packet.

GET_STATUS_UNIQUE

    A client issues this to get status information for a submitted job.

    Arguments:
    - Unique value that was given when job was submitted.

OPTION_REQ

    A client issues this to set an option for the connection in the
    job server. Returns a OPTION_RES packet on success, or an ERROR
    packet on failure.

    Arguments:
    - Name of the option to set. Possibilities are:
      * "exceptions" - Forward WORK_EXCEPTION packets to the client.


Client Responses
----------------

These response types may only be sent to a client:

JOB_CREATED

    This is sent in response to one of the SUBMIT_JOB* packets. It
    signifies to the client that a the server successfully received
    the job and queued it to be run by a worker.

    Arguments:
    - Job handle assigned by server.

WORK_DATA, WORK_WARNING, WORK_STATUS, WORK_COMPLETE,
WORK_FAIL, WORK_EXCEPTION

    For non-background jobs, the server forwards these packets from
    the worker to clients. See "Worker Requests" for more information
    and arguments.

STATUS_RES

    This is sent in response to a GET_STATUS request. This is used by
    clients that have submitted a job with SUBMIT_JOB_BG to see if the
    job has been completed, and if not, to get the percentage complete.

    Arguments:
    - NULL byte terminated job handle.
    - NULL byte terminated known status, this is 0 (false) or 1 (true).
    - NULL byte terminated running status, this is 0 (false) or 1
      (true).
    - NULL byte terminated percent complete numerator.
    - Percent complete denominator.

STATUS_RES_UNIQUE

    This is sent in response to a GET_STATUS_UNIQUE request. This is
    used by clients that have submitted a job with SUBMIT_JOB_BG to see
    if the job has been completed, and if not, to get the percentage
    complete.

    Arguments:
    - NULL byte terminated job handle.
    - NULL byte terminated known status, this is 0 (false) or 1 (true).
    - NULL byte terminated running status, this is 0 (false) or 1
      (true).
    - NULL byte terminated percent complete numerator.
    - NULL byte terminated percent complete denominator.
    - Count of clients waiting.

OPTION_RES

    Successful response to the OPTION_REQ request.

    Arguments:
    - Name of the option that was set, see OPTION_REQ for possibilities.


Worker Requests
---------------

These request types may only be sent by a worker:

CAN_DO

    This is sent to notify the server that the worker is able to
    perform the given function. The worker is then put on a list to be
    woken up whenever the job server receives a job for that function.

    Arguments:
    - Function name.

CAN_DO_TIMEOUT

     Same as CAN_DO, but with a timeout value on how long the job
     is allowed to run. After the timeout value, the job server will
     mark the job as failed and notify any listening clients.

     Arguments:
     - NULL byte terminated Function name.
     - Timeout value (in milliseconds).

CANT_DO

     This is sent to notify the server that the worker is no longer
     able to perform the given function.

     Arguments:
     - Function name.

RESET_ABILITIES

    This is sent to notify the server that the worker is no longer
    able to do any functions it previously registered with CAN_DO or
    CAN_DO_TIMEOUT.

    Arguments:
    - None.

PRE_SLEEP

    This is sent to notify the server that the worker is about to
    sleep, and that it should be woken up with a NOOP packet if a
    job comes in for a function the worker is able to perform.

    Arguments:
    - None.

GRAB_JOB

    This is sent to the server to request any available jobs on the
    queue. The server will respond with either NO_JOB or JOB_ASSIGN,
    depending on whether a job is available.

    Arguments:
    - None.

GRAB_JOB_UNIQ

    Just like GRAB_JOB, but return JOB_ASSIGN_UNIQ when there is a job.

    Arguments:
    - None.

GRAB_JOB_ALL

    Just like GRAB_JOB_UNIQ, but return JOB_ASSIGN_ALL when there is a job.

    Arguments:
    - None.

WORK_DATA

    This is sent to update the client with data from a running job. A
    worker should use this when it needs to send updates, send partial
    results, or flush data during long running jobs. It can also be
    used to break up a result so the worker does not need to buffer
    the entire result before sending in a WORK_COMPLETE packet.

    Arguments:
    - NULL byte terminated job handle.
    - Opaque data that is returned to the client.

WORK_WARNING

    This is sent to update the client with a warning. It acts just
    like a WORK_DATA response, but should be treated as a warning
    instead of normal response data.

    Arguments:
    - NULL byte terminated job handle.
    - Opaque data that is returned to the client.

WORK_STATUS

    This is sent to update the server (and any listening clients)
    of the status of a running job. The worker should send these
    periodically for long running jobs to update the percentage
    complete. The job server should store this information so a client
    who issued a background command may retrieve it later with a
    GET_STATUS request.

    Arguments:
    - NULL byte terminated job handle.
    - NULL byte terminated percent complete numerator.
    - Percent complete denominator.

WORK_COMPLETE

    This is to notify the server (and any listening clients) that
    the job completed successfully.

    Arguments:
    - NULL byte terminated job handle.
    - Opaque data that is returned to the client as a response.

WORK_FAIL

    This is to notify the server (and any listening clients) that
    the job failed.

    Arguments:
    - Job handle.

WORK_EXCEPTION

    This is to notify the server (and any listening clients) that
    the job failed with the given exception.

    Arguments:
    - NULL byte terminated job handle.
    - Opaque data that is returned to the client as an exception.

SET_CLIENT_ID

    This sets the worker ID in a job server so monitoring and reporting
    commands can uniquely identify the various workers, and different
    connections to job servers from the same worker.

    Arguments:
    - Unique string to identify the worker instance.

ALL_YOURS

    Not yet implemented. This looks like it is used to notify a job
    server that this is the only job server it is connected to, so
    a job can be given directly to this worker with a JOB_ASSIGN and
    no worker wake-up is required.

    Arguments:
    - None.


Worker Responses
----------------

These response types may only be sent to a worker:

NOOP

    This is used to wake up a sleeping worker so that it may grab a
    pending job.

    Arguments:
    - None.

NO_JOB

    This is given in response to a GRAB_JOB request to notify the
    worker there are no pending jobs that need to run.

    Arguments:
    - None.

JOB_ASSIGN

    This is given in response to a GRAB_JOB request to give the worker
    information needed to run the job. All communication about the
    job (such as status updates and completion response) should use
    the handle, and the worker should run the given function with
    the argument.

    Arguments:
    - NULL byte terminated job handle.
    - NULL byte terminated function name.
    - Opaque data that is given to the function as an argument.

JOB_ASSIGN_UNIQ

    This is given in response to a GRAB_JOB_UNIQ request and acts
    just like JOB_ASSIGN but with the client assigned unique ID.

    Arguments:
    - NULL byte terminated job handle.
    - NULL byte terminated function name.
    - NULL byte terminated unique ID.
    - Opaque data that is given to the function as an argument.

JOB_ASSIGN_ALL

    This is given in response to a GRAB_JOB_ALL request and acts
    just like JOB_ASSIGN_UNIQ but with the reducer returned.

    Arguments:
    - NULL byte terminated job handle.
    - NULL byte terminated function name.
    - NULL byte terminated unique ID.
    - NULL byte terminated reducer.
    - Opaque data that is given to the function as an argument.

Administrative Protocol
-----------------------

The Gearman job server also supports a text-based protocol to pull
information and run some administrative tasks. This runs on the same
port as the binary protocol, and the server differentiates between
the two by looking at the first character. If it is a NULL (\0),
then it is binary, if it is non-NULL, that it attempts to parse it
as a text command. The following commands are supported:

workers

    This sends back a list of all workers, their file descriptors,
    their IPs, their IDs, and a list of registered functions they can
    perform. The list is terminated with a line containing a single
    '.' (period). The format is:

    FD IP-ADDRESS CLIENT-ID : FUNCTION ...

    Arguments:
    - None.

status

    This sends back a list of all registered functions.  Next to
    each function is the number of jobs in the queue, the number of
    running jobs, and the number of capable workers. The columns are
    tab separated, and the list is terminated with a line containing
    a single '.' (period). The format is:

    FUNCTION\tTOTAL\tRUNNING\tAVAILABLE_WORKERS

    Arguments:
    - None.

prioritystatus

    This sends back a list of all registered functions. Next to each
    function is the number of queued jobs that are not running, broken down
    by priority, and the number of capable workers. The columns are tab
    separated, and the list is terminated with a line containing
    a single '.' (period). The format is:

    FUNCTION\tHIGH-QUEUED\tNORMAL-QUEUED\tLOW-QUEUED\tAVAILABLE_WORKERS

    Columns:
    - Function name.
    - Number of queued high priority jobs.
    - Number of queued normal priority jobs.
    - Number of queued low priority jobs.
    - Available workers registered for this function.

    Arguments:
    - None.

maxqueue

    This sets the maximum queue size for a function. If no size is
    given, the default is used. If one size is given, it is applied to
    jobs regardless of priority. If three sizes are given, the sizes
    are used when testing high-priority, normal, and low-priority jobs,
    respectively. A zero or negative size indicates no limit.  This
    command sends back a single line with "OK".

    Arguments:
    - Function name.
    - Optional maximum queue size (to apply one maximum at all priorities), or
      three optional maximum queue sizes (to enforce for high-, normal-, and
      low-priority job submissions).

version

    Send back the version of the server.

    Arguments:
    - None.


The Perl version also has a 'gladiator' command that uses the
'Devel::Gladiator' Perl module and is used for debugging.


Binary Protocol Example
-----------------------

This example will step through a simple interaction where a worker
connects and registers for a function named "reverse", the client
connects and submits a job for this function, and the worker performs
this job and responds with a result. This shows every byte that needs
to be sent over the wire in order for the job to be run to completion.


Worker registration:

Worker -> Job Server
00 52 45 51                \0REQ        (Magic)
00 00 00 01                1            (Packet type: CAN_DO)
00 00 00 07                7            (Packet length)
72 65 76 65 72 73 65       reverse      (Function)


Worker check for job:

Worker -> Job Server
00 52 45 51                \0REQ        (Magic)
00 00 00 09                9            (Packet type: GRAB_JOB)
00 00 00 00                0            (Packet length)

Job Server -> Worker
00 52 45 53                \0RES        (Magic)
00 00 00 0a                10           (Packet type: NO_JOB)
00 00 00 00                0            (Packet length)

Worker -> Job Server
00 52 45 51                \0REQ        (Magic)
00 00 00 04                4            (Packet type: PRE_SLEEP)
00 00 00 00                0            (Packet length)


Client job submission:

Client -> Job Server
00 52 45 51                \0REQ        (Magic)
00 00 00 07                7            (Packet type: SUBMIT_JOB)
00 00 00 0d                13           (Packet length)
72 65 76 65 72 73 65 00    reverse\0    (Function)
00                         \0           (Unique ID)
74 65 73 74                test         (Workload)

Job Server -> Client
00 52 45 53                \0RES        (Magic)
00 00 00 08                8            (Packet type: JOB_CREATED)
00 00 00 07                7            (Packet length)
48 3a 6c 61 70 3a 31       H:lap:1      (Job handle)


Worker wakeup:

Job Server -> Worker
00 52 45 53                \0RES        (Magic)
00 00 00 06                6            (Packet type: NOOP)
00 00 00 00                0            (Packet length)


Worker check for job:

Worker -> Job Server
00 52 45 51                \0REQ        (Magic)
00 00 00 09                9            (Packet type: GRAB_JOB)
00 00 00 00                0            (Packet length)

Job Server -> Worker
00 52 45 53                \0RES        (Magic)
00 00 00 0b                11           (Packet type: JOB_ASSIGN)
00 00 00 14                20           (Packet length)
48 3a 6c 61 70 3a 31 00    H:lap:1\0    (Job handle)
72 65 76 65 72 73 65 00    reverse\0    (Function)
74 65 73 74                test         (Workload)


Worker response for job:

Worker -> Job Server
00 52 45 51                \0REQ        (Magic)
00 00 00 0d                13           (Packet type: WORK_COMPLETE)
00 00 00 0c                12           (Packet length)
48 3a 6c 61 70 3a 31 00    H:lap:1\0    (Job handle)
74 73 65 74                tset         (Response)


Job server response to client:

Job Server -> Client
00 52 45 53                \0RES        (Magic)
00 00 00 0d                13           (Packet type: WORK_COMPLETE)
00 00 00 0c                12           (Packet length)
48 3a 6c 61 70 3a 31 00    H:lap:1\0    (Job handle)
74 73 65 74                tset         (Response)


At this point, the worker would then ask for more jobs to run (the
"Check for job" state above), and the client could submit more
jobs. Note that the client is full duplex and could have multiple
jobs being run over a single socket at the same time. The result
packets may not be sent in the same order the jobs were submitted
and instead interleaved with other job result packets.
{% endhighlight %}
