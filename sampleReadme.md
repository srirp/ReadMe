# CMS Audit Migration Testing Scenarios

## Overview
Comprehensive testing guide for Cassandra to Bigtable migration across all phases and column families.

**Testing Phases:**
1. **Phase 1:** Read Cassandra, Write Both (R_CASSANDRA + W_CASSANDRA_BIGTABLE)
2. **Phase 2:** Read Bigtable, Write Both (R_BIGTABLE + W_CASSANDRA_BIGTABLE)
3. **Phase 3:** Read Bigtable, Write Bigtable Only (R_BIGTABLE + W_BIGTABLE)
4. **Rollback:** Read Cassandra, Write Cassandra (R_CASSANDRA + W_CASSANDRA)

---

## Configuration Testing Matrix

### Table 1: Migration Phase Configurations

| Phase | Read Mode | Write Mode | Purpose | Risk Level |
|-------|-----------|------------|---------|------------|
| **Phase 1** | `R_CASSANDRA` | `W_CASSANDRA_BIGTABLE` | Dual write validation | **LOW** |
| **Phase 2** | `R_BIGTABLE` | `W_CASSANDRA_BIGTABLE` | Read path validation | **MEDIUM** |
| **Phase 3** | `R_BIGTABLE` | `W_BIGTABLE` | Full migration | **HIGH** |
| **Rollback** | `R_CASSANDRA` | `W_CASSANDRA` | Safety fallback | **LOW** |

### Table 2: Test Environment Setup

| Component | Configuration | Test Value |
|-----------|---------------|------------|
| **Cassandra Seeds** | `ccg13cmscas1.ccg13.slc.paypalinc.com:9160` | Verify connectivity |
| **Bigtable Project** | `ccg15-hrz-cms-platform` | Verify auth & permissions |
| **Bigtable Instance** | `pypl-cms` | Verify table creation |
| **Test Data** | All 6 column families | Use isolated test keys |

---

## Phase 1 Testing: Read Cassandra, Write Both

**Config:** `read_mode=R_CASSANDRA`, `write_mode=W_CASSANDRA_BIGTABLE`

### Test Scenario 1.1: New_History Table

#### Write Tests
```bash
# Test 1: Insert Operation - Standard Object
curl -X POST "http://localhost:8080/audit" \
  -H "Content-Type: application/json" \
  -d '{
    "repoName": "test-repo",
    "branchName": "test-branch",
    "clazzName": "TestServer",
    "oid": "test-obj-001",
    "operation": "Insert",
    "content": "{\"id\":\"test-obj-001\",\"name\":\"Test Object\"}"
  }'

# Expected: Write to BOTH Cassandra New_History AND Bigtable New_History
# Cassandra: Row="test-repo-test-branch-TestServer-test-obj-001", Column=timestamp
# Bigtable: Row="test-repo-test-branch-TestServer-test-obj-001#timestamp"
```

#### Read Tests
```bash
# Test 2: Read Operation - Verify Cassandra Read
curl "http://localhost:8080/audit/test-repo/test-branch/TestServer/test-obj-001?startTime=$(date -d '1 hour ago' +%s)000&format=object"

# Expected: Read from Cassandra New_History table only
# Verify response contains inserted data
```

#### Validation Checklist
- [ ] **Cassandra Write Success:** Check logs for "Data single write to Cassandra"
- [ ] **Bigtable Write Success:** Check logs for "Data single write to Bigtable"
- [ ] **Read from Cassandra:** Verify data returned matches inserted content
- [ ] **Dual Write Consistency:** Both databases contain identical data

### Test Scenario 1.2: History_class Table

#### Write Tests
```bash
# Same insert as 1.1 should also write to History_class
# Expected: Write to BOTH Cassandra History_class AND Bigtable History_class
# Cassandra: Row="timeBucket-test-repo-test-branch-TestServer", Column=timestamp, Value=objectId
# Bigtable: Row="timeBucket-test-repo-test-branch-TestServer#timestamp", Value=objectId
```

#### Read Tests
```bash
# Test 3: Class Query
curl "http://localhost:8080/audit/test-repo/test-branch/TestServer?startTime=$(date -d '1 hour ago' +%s)000&endTime=$(date +%s)000"

# Expected: Read from Cassandra History_class table only
```

#### Validation Checklist
- [ ] **Class Aggregation Written:** Both databases show class-level entry
- [ ] **Time Bucket Correct:** Verify proper time bucket calculation
- [ ] **Object ID Stored:** Value contains correct object ID

### Test Scenario 1.3: Delete Operations (Multiple Tables)

#### Write Tests
```bash
# Test 4: Delete Operation
curl -X POST "http://localhost:8080/audit" \
  -H "Content-Type: application/json" \
  -d '{
    "repoName": "test-repo",
    "branchName": "test-branch",
    "clazzName": "TestServer",
    "oid": "test-obj-delete",
    "operation": "Delete",
    "content": "{\"id\":\"test-obj-delete\",\"resourceId\":\"test-resource-001\"}"
  }'

# Expected: Write to 5 tables in BOTH databases:
# - New_History, History_class, New_History_deleted, New_History_deleted_reversed, Snapshot
```

#### Read Tests
```bash
# Test 5: Deleted Object Queries
curl "http://localhost:8080/audit/getDeletedObjectOids?repo=test-repo&branch=test-branch&class=TestServer&resourceId=test-resource-001"

curl "http://localhost:8080/audit/getDeletedResourceId?repo=test-repo&branch=test-branch&class=TestServer&oid=test-obj-delete"
```

#### Validation Checklist
- [ ] **All 5 Tables Written:** Cassandra and Bigtable both have entries in all tables
- [ ] **Deleted Object Tracking:** New_History_deleted contains correct data
- [ ] **Reverse Lookup:** New_History_deleted_reversed allows resource→object lookup
- [ ] **Snapshot Created:** Snapshot table has deletion record

### Test Scenario 1.4: ODB Operations

#### Write Tests
```bash
# Test 6: ODB Record
curl -X POST "http://localhost:8080/audit/odb" \
  -H "Content-Type: application/json" \
  -d '{
    "repoName": "test-repo",
    "branchName": "test-branch",
    "clazzName": "TestServer",
    "resourceId": "test-resource-odb",
    "content": "{\"odbData\":\"test-odb-content\"}"
  }'

# Expected: Write to BOTH ODB_History tables only
```

#### Read Tests
```bash
# Test 7: ODB Query
curl "http://localhost:8080/audit/odbHistory/test-repo/test-branch/TestServer/test-resource-odb?startTime=$(date -d '1 hour ago' +%s)000"

# Expected: Read from Cassandra ODB_History only
```

#### Validation Checklist
- [ ] **ODB Table Written:** Both databases have ODB_History entry
- [ ] **No Other Tables:** Verify only ODB_History was written
- [ ] **ODB Read Success:** Data retrieved correctly from Cassandra

---

## Phase 2 Testing: Read Bigtable, Write Both

**Config:** `read_mode=R_BIGTABLE`, `write_mode=W_CASSANDRA_BIGTABLE`

### Test Scenario 2.1: Read Path Validation

#### Pre-requisite Setup
```bash
# Ensure Phase 1 test data exists in both databases
# Change configuration to Phase 2 mode
```

#### Read Tests
```bash
# Test 8: Same queries as Phase 1, but reading from Bigtable
curl "http://localhost:8080/audit/test-repo/test-branch/TestServer/test-obj-001?startTime=$(date -d '1 hour ago' +%s)000&format=object"

# Expected: Read from Bigtable New_History (different row key format)
# Data should match Phase 1 results
```

#### Validation Checklist
- [ ] **Bigtable Read Success:** Logs show "query executed" for Bigtable
- [ ] **Data Consistency:** Response matches Phase 1 Cassandra read
- [ ] **Row Key Translation:** DaoWrapper correctly handles row key differences
- [ ] **No Cassandra Reads:** Verify no Cassandra queries in logs

### Test Scenario 2.2: Write Path Continues

#### Write Tests
```bash
# Test 9: New data in Phase 2
curl -X POST "http://localhost:8080/audit" \
  -H "Content-Type: application/json" \
  -d '{
    "repoName": "test-repo",
    "branchName": "test-branch",
    "clazzName": "TestServer",
    "oid": "test-obj-phase2",
    "operation": "Update",
    "content": "{\"id\":\"test-obj-phase2\",\"updated\":true}"
  }'

# Expected: Still write to BOTH databases
```

#### Read Tests
```bash
# Test 10: Read new data from Bigtable
curl "http://localhost:8080/audit/test-repo/test-branch/TestServer/test-obj-phase2?startTime=$(date -d '1 hour ago' +%s)000&format=object"

# Expected: Read from Bigtable, should see new data immediately
```

#### Validation Checklist
- [ ] **Dual Write Continues:** Both databases get new data
- [ ] **Bigtable Read Current:** New data immediately available from Bigtable
- [ ] **Cross-Phase Consistency:** Can read both Phase 1 and Phase 2 data

---

## Phase 3 Testing: Read Bigtable, Write Bigtable Only

**Config:** `read_mode=R_BIGTABLE`, `write_mode=W_BIGTABLE`

### Test Scenario 3.1: Bigtable-Only Operations

#### Write Tests
```bash
# Test 11: Write only to Bigtable
curl -X POST "http://localhost:8080/audit" \
  -H "Content-Type: application/json" \
  -d '{
    "repoName": "test-repo",
    "branchName": "test-branch",
    "clazzName": "TestServer",
    "oid": "test-obj-bigtable-only",
    "operation": "Insert",
    "content": "{\"id\":\"test-obj-bigtable-only\",\"bigtableOnly\":true}"
  }'

# Expected: Write to Bigtable ONLY (no Cassandra writes)
```

#### Read Tests
```bash
# Test 12: Read Bigtable-only data
curl "http://localhost:8080/audit/test-repo/test-branch/TestServer/test-obj-bigtable-only?startTime=$(date -d '1 hour ago' +%s)000&format=object"

# Expected: Successful read from Bigtable
```

#### Validation Checklist
- [ ] **No Cassandra Writes:** Logs show no "Data single write to Cassandra"
- [ ] **Bigtable Write Success:** Logs show "Data single write to Bigtable"
- [ ] **Read Success:** Data available immediately from Bigtable
- [ ] **Historical Data:** Can still read Phase 1 & 2 data from Bigtable

### Test Scenario 3.2: Performance Validation

#### Load Tests
```bash
# Test 13: Batch operations
curl -X POST "http://localhost:8080/audit/batch" \
  -H "Content-Type: application/json" \
  -d '[
    {"repoName": "test-repo", "branchName": "test-branch", "clazzName": "TestServer", "oid": "batch-001", "operation": "Insert", "content": "{\"batch\": true}"},
    {"repoName": "test-repo", "branchName": "test-branch", "clazzName": "TestServer", "oid": "batch-002", "operation": "Insert", "content": "{\"batch\": true}"},
    {"repoName": "test-repo", "branchName": "test-branch", "clazzName": "TestServer", "oid": "batch-003", "operation": "Insert", "content": "{\"batch\": true}"}
  ]'

# Expected: Efficient batch writes to Bigtable only
```

#### Validation Checklist
- [ ] **Batch Performance:** Compare response time vs Phase 1/2
- [ ] **No Dual Write Overhead:** Faster than dual-write phases
- [ ] **All Batch Items:** Verify all items written and readable

---

## Rollback Testing: Emergency Fallback

**Config:** `read_mode=R_CASSANDRA`, `write_mode=W_CASSANDRA`

### Test Scenario 4.1: Rollback Capability

#### Write Tests
```bash
# Test 14: Rollback write (Cassandra only)
curl -X POST "http://localhost:8080/audit" \
  -H "Content-Type: application/json" \
  -d '{
    "repoName": "test-repo",
    "branchName": "test-branch",
    "clazzName": "TestServer",
    "oid": "test-obj-rollback",
    "operation": "Insert",
    "content": "{\"rollback\": true}"
  }'

# Expected: Write to Cassandra ONLY
```

#### Read Tests
```bash
# Test 15: Read from Cassandra in rollback mode
curl "http://localhost:8080/audit/test-repo/test-branch/TestServer/test-obj-rollback?startTime=$(date -d '1 hour ago' +%s)000&format=object"

# Expected: Read from Cassandra, should see rollback data
```

#### Validation Checklist
- [ ] **Cassandra Only:** No Bigtable writes in logs
- [ ] **Read Historical:** All historical data still accessible
- [ ] **New Data Written:** Rollback data properly stored in Cassandra
- [ ] **System Stability:** No errors or degradation

---

## Cross-Phase Data Consistency Testing

### Test Scenario 5.1: Data Migration Validation

#### Consistency Tests
```bash
# Test 16: Compare data between databases (manual validation)
# Use test endpoints to read same data from both databases

# Read from Cassandra directly
curl "http://localhost:8080/audit/dual-write-test/read-cassandra/test-repo-test-branch-TestServer-test-obj-001"

# Read from Bigtable directly
curl "http://localhost:8080/audit/dual-write-test/read-bigtable/test-repo-test-branch-TestServer-test-obj-001#<timestamp>"
```

#### Validation Checklist
- [ ] **Content Match:** JSON content identical between databases
- [ ] **Timestamp Consistency:** Timestamps match (accounting for format differences)
- [ ] **All Test Objects:** Every test object exists in both databases
- [ ] **No Data Loss:** Count of records matches between databases

### Test Scenario 5.2: Range Query Testing

#### Complex Query Tests
```bash
# Test 17: Large time range queries
curl "http://localhost:8080/audit/test-repo/test-branch/TestServer/test-obj-001?startTime=$(date -d '2 hours ago' +%s)000&endTime=$(date +%s)000&format=object"

# Test 18: Class-level aggregation
curl "http://localhost:8080/audit/test-repo/test-branch/TestServer?startTime=$(date -d '2 hours ago' +%s)000&endTime=$(date +%s)000"

# Test 19: Pagination
curl "http://localhost:8080/audit/test-repo/test-branch/TestServer/test-obj-001?startTime=$(date -d '2 hours ago' +%s)000&pageSize=5&skip=0"
```

#### Validation Checklist
- [ ] **Range Query Accuracy:** Results consistent across phases
- [ ] **Pagination Works:** Proper page boundaries and ordering
- [ ] **Class Aggregation:** Correct object counts and time buckets
- [ ] **Performance Acceptable:** Response times within SLA

---

## Error Handling & Edge Cases

### Test Scenario 6.1: Error Conditions

#### Database Failure Simulation
```bash
# Test 20: Simulate Cassandra unavailable (Phase 1)
# Stop Cassandra temporarily
curl -X POST "http://localhost:8080/audit" \
  -H "Content-Type: application/json" \
  -d '{"repoName": "test-repo", "branchName": "test-branch", "clazzName": "TestServer", "oid": "error-test", "operation": "Insert", "content": "{\"error\": \"test\"}"}'

# Expected: Should handle gracefully, continue to Bigtable
```

#### Validation Checklist
- [ ] **Graceful Degradation:** System continues operating
- [ ] **Error Logging:** Appropriate error messages logged
- [ ] **Recovery Records:** MongoDB recovery records created
- [ ] **Service Availability:** API remains responsive

### Test Scenario 6.2: Data Type Edge Cases

#### Special Character Tests
```bash
# Test 21: Special characters in data
curl -X POST "http://localhost:8080/audit" \
  -H "Content-Type: application/json" \
  -d '{
    "repoName": "test-repo",
    "branchName": "test-branch",
    "clazzName": "TestServer",
    "oid": "special-chars-#-test",
    "operation": "Insert",
    "content": "{\"special\": \"data with #, -, and unicode: 你好\"}"
  }'

# Expected: Handle special characters properly in both databases
```

#### Validation Checklist
- [ ] **Special Characters:** Properly encoded/decoded
- [ ] **Row Key Safety:** Hash separator doesn't conflict with data
- [ ] **Unicode Support:** International characters preserved
- [ ] **JSON Escaping:** Proper JSON encoding maintained

---

## Performance Benchmarking

### Test Scenario 7.1: Load Testing

#### Performance Test Matrix

| Test Type | Phase 1 | Phase 2 | Phase 3 | Expected Improvement |
|-----------|---------|---------|---------|---------------------|
| **Single Write** | Dual overhead | Dual overhead | Bigtable only | **50% faster** |
| **Batch Write** | Dual overhead | Dual overhead | Bigtable only | **60% faster** |
| **Single Read** | Cassandra | Bigtable | Bigtable | **Similar or better** |
| **Range Query** | Cassandra | Bigtable | Bigtable | **Varies by query** |

#### Load Test Commands
```bash
# Test 22: Concurrent writes (use concurrent curl or load testing tool)
for i in {1..100}; do
  curl -X POST "http://localhost:8080/audit" \
    -H "Content-Type: application/json" \
    -d "{\"repoName\": \"test-repo\", \"branchName\": \"test-branch\", \"clazzName\": \"TestServer\", \"oid\": \"load-test-$i\", \"operation\": \"Insert\", \"content\": \"{\\\"loadTest\\\": $i}\"}" &
done
wait

# Test 23: Concurrent reads
for i in {1..50}; do
  curl "http://localhost:8080/audit/test-repo/test-branch/TestServer/load-test-$i?startTime=$(date -d '1 hour ago' +%s)000" &
done
wait
```

#### Performance Validation
- [ ] **Write Throughput:** Measure ops/second in each phase
- [ ] **Read Latency:** P95 response times acceptable
- [ ] **Error Rate:** <1% errors under load
- [ ] **Resource Usage:** CPU/memory within limits

---

## Testing Completion Checklist

### Phase 1 Completion
- [ ] All 6 column families write to both databases
- [ ] All read operations use Cassandra
- [ ] Data consistency verified
- [ ] Performance baseline established

### Phase 2 Completion
- [ ] All read operations use Bigtable
- [ ] Write operations continue to both databases
- [ ] Cross-phase data accessible
- [ ] No functional regressions

### Phase 3 Completion
- [ ] All operations use Bigtable only
- [ ] Performance improved over dual-write phases
- [ ] All historical data accessible
- [ ] System fully migrated

### Rollback Validation
- [ ] Emergency rollback tested successfully
- [ ] All functionality works in rollback mode
- [ ] No data loss during rollback
- [ ] Confidence in safety mechanism

### Final Sign-off
- [ ] All test scenarios completed
- [ ] Performance meets requirements
- [ ] Error handling validated
- [ ] Data consistency confirmed
- [ ] Documentation updated
- [ ] Monitoring alerts configured

## Test Data Cleanup

```bash
# Clean up test data after all testing
# Use pattern matching to remove test-* entries from both databases
curl -X DELETE "http://localhost:8080/audit/cleanup/test-repo/test-branch/TestServer/*"
```

**🎯 Testing Complete:** System ready for production migration!
