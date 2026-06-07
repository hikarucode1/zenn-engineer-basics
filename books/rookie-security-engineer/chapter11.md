---
title: "第11章 ブルーチーム ── SOC / CSIRT / インシデント対応"
---

Part 4では、いよいよ現場の職種を一つずつ覗いていきます。まずは **守る側**、ブルーチームの世界です。

## SOC と CSIRT の役割

しばしば混同される2つのチームです。

- **SOC (Security Operation Center)**: **平時の監視** が中心。アラートを24時間体制でトリアージし、脅威を検知する
- **CSIRT (Computer Security Incident Response Team)**: **有事の対応** が中心。インシデント発生時に調査・封じ込め・復旧を主導する

SOCが「常駐の警備員」、CSIRTが「招集される消防隊」というイメージです。組織によっては両者を兼ねていることもあります。

## ログ・SIEM・EDR

ブルーチームが日常的に触るツールが、SIEMとEDRです。

- **SIEM (Security Information and Event Management)**: 各種ログを一元集約し、ルールやAIで異常を検知するプラットフォーム。Splunk、Microsoft Sentinel、Elastic Securityなど
- **EDR (Endpoint Detection and Response)**: PCやサーバーの端末側で挙動を監視・遮断するツール。CrowdStrike、SentinelOne、Microsoft Defender for Endpointなど

SOCアナリストの仕事の多くは「SIEMが上げたアラートをEDRやログで深掘りし、本物の脅威か誤検知かを判断する」ことです。

## インシデント対応プロセス

事実上の業界標準が **NIST SP 800-61** の4フェーズです。

1. **準備 (Preparation)**: 体制・手順・ツールを整備
2. **検知と分析 (Detection & Analysis)**: 何が起きているかを把握
3. **封じ込め・根絶・復旧 (Containment, Eradication & Recovery)**: 被害拡大を止め、原因を取り除き、業務を戻す
4. **事後活動 (Post-Incident Activity)**: 再発防止と振り返り

特に重要なのが **準備** と **事後活動** です。インシデントの現場対応力は、平時の備えと前回の教訓で9割が決まります。

## 検知エンジニアリングという世界

最近注目されている専門職が **検知エンジニア (Detection Engineer)** です。

新しい攻撃手法に対する検知ルールを設計・テスト・運用する仕事で、攻撃者の手口(ATT&CKフレームワークなど)を読み解き、自社環境のログにマッピングし、誤検知を抑えながら本物の脅威だけを拾えるルールを書きます。SOCアナリストから一段上のキャリアパスとして人気が高まっています。

## 必要なスキルセット

ブルーチームで活躍するために重要なスキルを挙げると、

- **ログを読む執念**: 1日に何万行も流れるログから異常を見つけ出す根気
- **ネットワーク・OSの知識**: パケットやプロセス挙動の意味が分かること
- **クエリ言語**: SPL (Splunk)、KQL (Microsoft Sentinel)、ESQL (Elastic) など
- **スクリプティング**: Python、PowerShell でデータ加工や自動化
- **報告書作成**: 経営層・顧客にも伝わる文章力

「派手さはないが奥が深い」典型的な仕事です。地道にログと向き合うのが好きな人には天職になります。
