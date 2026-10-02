# Validation: splunk01 Monitoring Pipeline

| Check | Expected | Result |
|---|---|---|
| Splunk ports | 8000, 8089, 9997 listening, owned by splunkd | PASS |
| Splunk survives reboot | Splunkd enabled | <PASS/FAIL> |
| Host firewall | Only required ports, scoped by source | <PASS/FAIL> |
| Forwarder enabled at boot | `systemctl is-enabled SplunkForwarder` = enabled | PASS |
| Forwarder can read root-only logs | `sudo -u splunkfwd head` succeeds via ACL | PASS |
| Forwarder connected | ESTAB to 192.168.20.30:9997 | <PASS/FAIL> |
| rhel01 events indexed | secure, messages, httpd logs in `index=linux` | PASS |
| Synthetic check | One event per minute, status 200, state UP | PASS |
| Alert saved | Scheduled every minute, triggers on non-UP | PASS |

## Output
```
<paste: ss -tlnp | grep -E '8000|8089|9997'>
```
```
<paste: sudo ufw status numbered>
```
```
<paste: getfacl /var/log/secure  and  sudo -u splunkfwd head -2 /var/log/secure>
```
```
<paste: sudo ss -tnp | grep 9997 on rhel01>
```
```
<paste: tail -3 /var/log/svccheck/rhel01-http.log>
```