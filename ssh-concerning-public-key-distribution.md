Part I : the importance of ssh public key distribution
======================================================

This is important.


Part II : examples of public-key distribution
=============================================

version 1: (on slack) "Hey everyone! I just rolled our web-servers! The new public-keys are: "

Software support:
    * script to push keys to local ssh client.
    * (what about case of non-local ssh client? i.e. unified cloud entry layup point).

version 2: push to ec2-instance metadata (instance meta-data is accessible through ec2 command line tools which, in turn, use an https authenticated rest interface. So were bootstrapping our public-key distribution using the fact we already have amazons public key.

Software support:
    * script to push ssh public key to ec2 instance meta-data
    * script to ping that meta-data during connection establishment (needs to be hooked into your connection establishment work-flow)

this should be enough for 95% of the environments out there.

