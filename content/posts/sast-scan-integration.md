+++
title = "安全性靜態掃描工具 sast-scan 與 Bitbucket 整合"
description = "ShiftLeft sast-scan VS SonarQube"
date = 2021-09-18T20:08:30+08:00
tags = ["shell", "scan", "security", "bitbucket", "sonarqube", "sast-scan", "sast", "sca"]
draft = false
+++

安全性掃描一直以來都是有點微妙的存在，之前的公司規定專案要上線要過 Fortify + Blackduck 的掃描，可是這類掃描都得花一點時間，所以就跟跑很慢的 Test Suite 一樣，總是淪落為有需要交差的時候才跑，然後每次掃完都得另外撥出時間來修復被掃到的弱點。

這回又遇上一樣的需求，只是這次有機會可以從頭評估，趁這這個勢頭，研究一下有沒有更好整進開發週期，掃描不太慢結果也不差的方案，是不是就能某種程度上趨近「越痛的事情就要越常做才會慢慢不痛」，展開尋找與探索之旅。

## TL;DR

1. [sast-scan](https://github.com/ShiftLeftSecurity/sast-scan) 跟 [SonarQube](https://www.sonarqube.org/) 掃出來的結果，安全性項目更多，煩人的假警報較少，掃描大專案的時候比較快。
2. 使用 [sast-scan](https://github.com/ShiftLeftSecurity/sast-scan) 的 Docker image 配合客製化的 bitbucket pipe 執行掃描結果判讀，可以實現~~不用花大錢~~就能提高系統安全性信心，而且還能滿足初步的 Rule 客製化的需求。

## sast-scan 與 SonarQube

實際上拿了一些專案來掃，幾個發現記錄一下：

1. SonarQube 跟 sast-scan 掃出來的東西用檔案名稱跟行數去對，重疊率在 5% 以下
2. SonarQube 的預設掃描規則，有一大部分是類似 Linter 的 Code Smell 項目
3. SonarQube 的 Docker image 跑在 local 的話 Docker 的記憶體不給高一點會爆，因為 SonarQube 的 embed ElasticSearch DB 需要至少 2G 的資源來跑，後來用 `docker run -d -m 8G --name sonarqube -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true -p 9000:9000 sonarqube:8.4.2-community` 去克服部分的專案在我本機會停住的瓶頸，但還是會有一些專案再怎麼加資源都會跑到停住懷疑自己。
4. SonarQube 掃完後的結果是留在 SonarQube 的 instance DB 裡，但我是要把 scan tool 在本機掃完以後報告拉出來集中解讀，SonarQube 報告要匯出成檔案的話得另外裝工具，找到 [sonar-cnes-report](https://community.sonarsource.com/t/new-release-sonarqube-cnes-report-3-3-1/41329/16) 可以匯出，但 SonarQube 兼容最高版本是 `8.4.2`。
5. SonarQube 8.4.2 沒有內建 library risk 的掃描。
6. SonarQube 的 Test Coverage 基本上是吃各家語言的 coverage xml 的路徑到指定的參數。需要看測試涵蓋率數字的話，也是得各專案先能產出 coverage xml 並正確送給 SonarQube 的 [Scanner 參數](https://docs.sonarqube.org/latest/analysis/coverage/)。
7. sast-scan 的 dependency scan 只會偵測到 root folder 的相關 package file，所以如果專案有 package 定義檔不在 root folder 的話，會需要到不同的 folder 去做掃描。

## 採取的策略跟導入專案前的準備

綜合了這些發現，初期就以最小限度客製化 sast-scan 的方式來切入，幾件事情需要解決：

1. 提供 best practice 給夥伴參考，CI 整合越簡單越好
2. 本機開發環境也可以看得到的掃描結果
3. Code Review 的時候很方便可以看到哪邊 fail 以及嚴重級別等資訊
4. 真的掃出太多沒辦法一次解完的時候，專案可以個別調整是否放寬或延後修復的時機
5. 有個機制可以統整所有專案的狀態

使用的工具：

1. bitbucket-pipeline.yml
2. [custom bitbucket pipe](https://bitbucket.org/atlassian/demo-pipe-python)

sast-scan 提供了[整合 Github 的範例](https://slscan.io/en/latest/getting-started/use-cases/)可以參考，另外有 [Bitbucket Pipeline](https://slscan.io/en/latest/integrations/bitbucket/) 的整合，只是不知為何 report 跟 PR comment 的部分好像是得跟 ShiftLeft SaaS 的服務綁一起才能用，所以 Bitbucket code insight 的 Report / Annotation 我就自己手工藝了。

雖然這個小工具不複雜，但實際程式碼不方便直接公開，倒是有幾個實作上遇到的點可以分享一下：

1. [custom bitbucket pipe](https://bitbucket.org/atlassian/demo-pipe-python) 裡面使用的出版套件 `semversioner` 版本是 `0.8.1 ` 如果開發環境用的版本比較高不知為何套用到範例裡的 pipeline 會一直噴錯，本機的套件安裝建議用 `pip install -Iv semversioner==0.8.1`
2. 因為 dependency scan 有可能需要在不同的資料夾下跑才會搜集完全，但是一般的 Analyzer 卻又可以偵測所有檔案，所以會出現集結出來的 sarif 檔案會有重複，在拉 finding 的時候要注意同一個地方找到的同一件事情只能算一次。
3. 我為了要可以客製化 Quality Gate 的標準，所以讓 pipe.py 可以吃 variable 參數來放寬判定 step fail 的閥值
4. sast-scan 目前只看到 [inline suppression](https://slscan.io/en/latest/getting-started/#suppression) 可以用，如果要整個檔案或者是 package 要排除不算 finding count 的話，我目前是暴力解，讓 pipe.py 吃 regex 參數進來，在爬 sarif/depscan 掃描結果的時候去 match regex string 來做排除。
5. SARIF 檔裡的 file location 是 bitbuckt mount 進 container 的絕對路徑，不能直接當參數塞給 bitbucket 的 report annotation，要先換成相對路徑才能組成對的連結在 Bitbucket UI 上。

## 後記

SARIF 檔案是靜態掃描工具的通用 protocol，微軟有個[簡介](https://github.com/microsoft/sarif-tutorials)說明得挺詳細的，這個 parsing 的操作可能有機會可以變成一個獨立的小工具，例如聚合一堆 SARIF 做正規化之類的處理。

之後如果要把所有專案的掃描結果通通送到中央廚房來料理的話，或許有個通用的工具也是不錯的發展。
