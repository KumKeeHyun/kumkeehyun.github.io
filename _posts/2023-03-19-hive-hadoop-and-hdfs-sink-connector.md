---
title: "빅데이터프로그래밍 프로젝트 3편"
subtitle: "hdfs-sink-connector 테스트하기"
permalink: posts/bdp-project/test-hdfs-sink-connector
classes: wide

categories:
  - bdp project
tags:
  - hive
  - hdfs
  - kafka connect
last_modified_at: 2023-03-19T00:00:00-00:00
---

# TOC
<!--ts-->
- [TOC](#toc)
- [서론](#서론)
- [hive, hdfs 구축](#hive-hdfs-구축)
- [이슈](#이슈)
  - [Json Convertor Schema](#json-convertor-schema)
  - [Decimal](#decimal)
- [결과](#결과)
<!--te-->

# 서론

작성중

# hive, hdfs 구축

# 이슈

## Json Convertor Schema

테스트라 `format.class`를 Json으로 했었음 -> 안됨

[json convertor with schema](https://github.com/confluentinc/kafka-connect-hdfs/issues/373)

## Decimal

<img width="1336" alt="image" src="https://user-images.githubusercontent.com/44857109/226175388-3304d9a4-453d-4514-a95f-02eedcd922a8.png">

[hive decimal precision 38](https://github.com/confluentinc/kafka-connect-hdfs/issues/279)

그런데 kafka connect decimal이 precision 설정을 지원하지 않음

# 결과

<img width="1341" alt="image" src="https://user-images.githubusercontent.com/44857109/226174678-84cede37-62e1-483e-9478-d9ab1b8ee629.png">

<img width="1338" alt="image" src="https://user-images.githubusercontent.com/44857109/226174602-48eb6f28-3f29-4f51-8e48-e338628fdf46.png">