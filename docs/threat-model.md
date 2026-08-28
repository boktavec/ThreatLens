# Threat Model

## Scope

This will be a local, docker compose based lab. Portable so anyone with docker compose can run it. We will simulate auth logs as well as collect container telemetry (container logs). Tools we will use:

- Fluent Bit -> to collect the data
- Opensearch -> to store and search
- Dashboards to investigate
  Will have end to end detection with repeated failed ssh authentication from a single source within a specific time window. This will be yaml definition is evaluated by a python detection runner. Alerts will be stored in OpenSearch - no notifications/response actions. All behavior (malicious and good) is local and deliberately simulated

## Assets to Protect

We will protect simulated user accounts and authentication access. An attacker that gains access can enter the simulated Linux environment and potentially access sensitive data.

We will protect simulated workload data and fake credentials/secrets. A breach of this data would simulate an attacker accessing sensitive information.

We will protect the local container runtime and workloads from unauthorized access. An attacker could run unapproved processes, alter workloads, or cause a disruption.

We will protect the integrity of collected telemetry. Missing, altered, or malformed events can create blind spots, cause missed detections, and lead to unreliable investigation results.

## Threat Actors

An external attacker will attempt to access the simulated Linux environment from an external IP address. They will try password guessing against a single user account to gain access. Their goal is to gain access to the environment and reach fake credentials, secrets, and workload data.

This attacker is limited to the local lab. They will not attack real systems, use privilege escalation, exploit vulnerabilities, move laterally, or alter telemetry.

A CI/automation identity will have valid access to run approved containers. This identity will misuse an approved workload in an unusual way that causes it to log access to fake secrets or workload data.

This identity will not modify Fluent Bit, OpenSearch, detection definitions, Docker runtime configuration, or systems outside the local lab.

## Behavior to Simulate

### Repeated failed SSH authentication

We will simulate 10 failed SSH authentication attempts from a single
source IP against a single user account within 60 seconds. This will
simulate an external attacker trying to guess a password.

Telemetry needed:

- timestamp
- authentication outcome
- username
- source IP
- host/service

Legitimate lookalikes:

- A user repeatedly enters an outdated password from a terminal or
  SSH client.
- A CI/deployment job still uses an expired or rotated credential.

Detection limitations:

- An attacker can spread attempts across multiple IPs or stay below
  the threshold.
- Fluent Bit can miss or delay log events, or fields can be
  malformed.

### Failed SSH authentication followed by success

We will simulate 3 failed SSH authentication attempts followed by a
successful SSH login from the same source IP and user account within 2
hours. This could show that an attacker was eventually able to guess
or gain access to valid credentials.

Telemetry needed:

- timestamp
- authentication outcome
- username
- source IP
- host/service
- process ID as optional investigation context

Legitimate lookalikes:

- A user enters an outdated password several times and then
  remembers or resets the correct password.
- A CI/deployment job uses a stale credential and succeeds after the
  credential is rotated or fixed.

Detection limitations:

- An attacker with valid credentials will not create failed login
  events.
- Shared NAT or VPN IP addresses can make separate users look like
  one sequence.
- This behavior is suspicious but does not prove that an attacker
  gained access.

### Unusual secret access by CI/automation identity

We will simulate a CI/automation identity using an approved workload
to access a fake `/secrets` endpoint. The workload will create a
structured container log event when secret access happens. This is
suspicious because the CI/automation identity should not normally
access secrets.

Telemetry needed:

- timestamp
- container name or ID
- workload/service name
- identity that made the request
- action or endpoint requested
- resource accessed
- outcome/status

Legitimate lookalikes:

- A deployment job retrieves an expected secret during startup.
- An engineer runs an approved debugging workflow during an
  incident.

Detection limitations:

- This will not be detected if the workload does not log the secret
  access.
- An attacker could access data through another path that does not
  create this event.
- Application log identity fields should be treated as context and
  not proof of identity.
