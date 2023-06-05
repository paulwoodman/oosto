# Oosto v2 migration playbook to upgrade servers in Mendards environment

To be downloaded to the local Satellite machine which will have access to all the store servers.

Playbook needs to be run in two stages.

The first playbook backs up all v1 watchlist camera data (Not the application) to the /storage partition. Once the backup is completed all store machine will be updated to RHEL 7.9. Any store machine that is running RHEL workstation will be migrated over to RHEL server. Once the update to RHEL 7.9 is completed the playbook will prep and upgrade the stoe machine to RHEL 8 which is requred for Oosto v2


