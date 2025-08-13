# Kibana Alert Rules (Contoh KQL)

## 1) Brute Force Spike
Query: event.dataset: cowrie* AND (message: "login failed" OR cowrie.login.failed: *)
Threshold: > 50 events in 5 minutes

## 2) File Drop Attempt
Query: event.dataset: cowrie* AND message: /wget|curl/

## 3) ICS Write Detected
Query: event.dataset: conpot* AND message: /(WriteSingleRegister|WriteMultipleRegisters)/