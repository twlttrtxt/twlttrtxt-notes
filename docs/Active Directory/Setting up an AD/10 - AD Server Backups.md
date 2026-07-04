This enables full system recovery in the case of hardware failure or else.

To install this backup functionality, go to server manager and then:

1. Go to Manage -> Add roles and features
2. Select features
3. Add Windows Server Backup
4. Complete installation.

Backups should be stored on separate storage devices such as external hard drives or dedicated backup servers, as that ensures that the backups remain available if the primary system fails.

A Bare-Metal Recovery backup includes the whole system state, includeing operating system, roles, configuration, and state data. This allows for complete system recovery in the case of hardware failure.