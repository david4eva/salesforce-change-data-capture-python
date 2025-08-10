# Salesforce Change Data Capture Python Client

A robust Python client for subscribing to Salesforce Change Data Capture (CDC) events using the Pub/Sub API. Eliminates race conditions and provides real-time data synchronization without the complexity of polling or record locking.

[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![gRPC](https://img.shields.io/badge/gRPC-1.50+-green.svg)](https://grpc.io/)

## 🚨 The Problem

Traditional Salesforce data synchronization approaches suffer from critical issues:

### Common Anti-Patterns:
- **Checkbox flags** to mark "processed" records
- **Batch processes** polling every 5-15 minutes
- **REST/SOAP APIs** with complex retry logic
- **Record locking** with `FOR UPDATE` causing `UNABLE_TO_LOCK_ROW` exceptions

### The Pain Points:
- ❌ Race conditions between multiple processes
- ❌ Data inconsistency and lost updates
- ❌ Polling delays and missed changes
- ❌ Complex error handling and retry logic
- ❌ Performance degradation from lock contention
- ❌ Constant debugging of integration failures

## ✅ The Solution: Change Data Capture

CDC provides a modern, event-driven approach to data synchronization:

- 🚀 **Real-time updates** (sub-second latency)
- 🔄 **Multiple simultaneous subscribers**
- 📋 **Guaranteed event ordering** with sequence numbers
- 🔄 **Built-in replay capability** with replay IDs
- 🚫 **No polling or locking required**
- 🛡️ **Automatic failure recovery**

## 🏗️ Architecture

```
Salesforce → Change Data Capture → Pub/Sub API → Python Client → Your Application
     ↓              ↓                    ↓             ↓              ↓
   Record      Event Stream         gRPC Stream    Event Handler   Business Logic
   Changes      (Real-time)         (Reliable)     (Ordered)       (Your Code)
```

## 🚀 Quick Start

Prerequisites

- Python 3.8+
- Salesforce org with Change Data Capture enabled
- Valid Salesforce credentials

### Setup Guide: 
[Python Quick Start for Pub/Sub API](https://developer.salesforce.com/docs/platform/pub-sub-api/guide/qs-python-quick-start.html)


## 📊 Performance Considerations

### Throughput Guidelines

| Scenario | Recommended Settings | Expected Performance |
|----------|---------------------|---------------------|
| Low volume (< 1000 events/day) | batch_size: 1 | Real-time processing |
| Medium volume (1000-10000 events/day) | batch_size: 10 | < 1 second latency |
| High volume (> 10000 events/day) | batch_size: 50 | < 5 second latency |

## 🙏 Acknowledgments

- [Salesforce](https://developer.salesforce.com/docs/platform/pub-sub-api/guide/intro.html) for the Pub/Sub API and excellent documentation
- [gRPC](https://grpc.io/) for the robust communication framework
- [Trailhead Change Data Capture Module](https://trailhead.salesforce.com/content/learn/modules/change-data-capture) for practical CDC implementation guidance
- The Salesforce developer community for inspiration and feedback

**⭐ If this project helps you, please consider starring it on GitHub!**

Made with ❤️ by the Salesforce developer community
