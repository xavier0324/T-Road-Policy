# T-Road: Policy



## Version history

## Table of Contents

<!-- toc -->

- [License](#license)
- [1 Introduction](#1-introduction)
  * [1.1 Overview](#11-overview)
  * [1.2 Terms and Abbreviations](#12-terms-and-abbreviations)
  * [1.3 References](#13-references)
- [2 Component View](#2-component-view)
  * [2.1 Web Server](#21-web-server)
  * [2.2 Configuration Processor](#22-configuration-processor)
  * [2.3 Signer](#23-signer)
  * [2.4 Configuration Client](#24-configuration-client)
  * [2.5 Password Store](#25-password-store)
  * [2.6 SSCD](#26-sscd)
- [3 Interfaces](#3-interfaces)
  * [3.1 Downloading Configuration](#31-downloading-configuration)
- [4 Configuration Distribution Workflow](#4-configuration-distribution-workflow)
- [5 Deployment View](#5-deployment-view)

<!-- tocstop -->

## License

This document is licensed under the Creative Commons Attribution-ShareAlike 3.0 Unported License. To view a copy of this license, visit http://creativecommons.org/licenses/by-sa/3.0/

## 1 Introduction

This document describes the architecture of the X-Road configuration proxy. For more information about X-Road and the role of the configuration proxy see \[[ARC-G](#Ref_ARC-G)\].

This document presents an overview of the components of the configuration proxy and the interfaces between these components. It is aimed at technical readers who want to acquire an overview of inner workings of the configuration proxy.


### 1.1 Overview

The configuration proxy acts as an intermediary between X-Road servers in the matters of global configuration exchange.

The goal of the configuration proxy is to download an X-Road global configuration from a provided configuration source and further distribute it in a secure way. Optionally, the downloaded global configuration may be modified to suit the requirements of the configuration proxy owner.

The configuration proxy can be configured to mediate several global configurations (from multiple configuration sources).


### 1.2 Terms and Abbreviations

See X-Road terms and abbreviations documentation \[[TA-TERMS](#Ref_TERMS)\].


### 1.3 References

1. <a id="Ref_ARC-G" class="anchor"></a>\[ARC-G\] Cybernetica AS. X-Road Architecture. Document ID: [ARC-G](arc-g_x-road_arhitecture.md).

2. <a id="Ref_PR-GCONF" class="anchor"></a>\[PR-GCONF\] Cybernetica AS. X-Road: Protocol for Downloading Configuration. Document ID: [PR-GCONF](../Protocols/pr-gconf_x-road_protocol_for_downloading_configuration.md).

3. <a id="Ref_PKCS11" class="anchor"></a>\[PKCS11\] Cryptographic Token Interface Standard. RSA Laboratories, PKCS\#11.

4. <a id="Ref_UG-CP" class="anchor"></a>\[UG-CP\] Cybernetica AS. X-Road v6 Configuration Proxy Manual. Document ID: [UG-CP](../Manuals/ug-cp_x-road_v6_configuration_proxy_manual.md).

5. <a id="Ref_ARC-TEC" class="anchor"></a>\[ARC-TEC\] X-Road technologies. Document ID: [ARC-TEC](arc-tec_x-road_technologies.md).

6. <a id="Ref_TERMS" class="anchor"></a>\[TA-TERMS\] X-Road Terms and Abbreviations. Document ID: [TA-TERMS](../terms_x-road_docs.md).

## 2 Component View

[Figure 1](#Ref_Configuration_proxy_component_diagram) shows the main components and interfaces of the X-Road configuration proxy. The components and the interfaces are described in detail in the following sections.


<a id="Ref_Configuration_proxy_component_diagram" class="anchor"></a>
![](img/arc-cp_configuration_proxy_component_diagram.png)

Figure 1. Configuration proxy component diagram

Technologies used in the configuration proxy can be found here: \[[ARC-TEC](#Ref_ARC-TEC)\]

### 2.1 Web Server

The global configuration files the configuration proxy generates need to be made available to configuration clients. The HTTP-based protocol used for downloading configuration is described in \[[PR-GCONF](#Ref_PR-GCONF)\]. Technically, the configuration consists of a set of files that are shared out using standard web server (nginx\[[1](#Ref_1)\] web server is used). The configuration processor component prepares and signs the configuration files and then copies them to the web server's output directory where they are distributed via standard means. See [Section 3.1](#31-downloading-configuration) for details on configuration distribution.


<a id="Ref_1" class="anchor"></a>
\[1\] See [*http://nginx.org/*](http://nginx.org/) for details.


### 2.2 Configuration Processor

A configuration processor is responsible for initiating the download of global configuration files for the configured X-Road instance and preparing them for distribution to configuration clients.

A cron job configured on the server periodically (every minute) starts the configuration processor, which sequentially performs it's functions for all configured configuration proxy instances.


### 2.3 Signer

The signer component is responsible for managing the keys and certificates used for signing the global configuration. Signer is called from the configuration processor to create the signature for the configuration.


### 2.4 Configuration Client

The configuration client is responsible for downloading remote global configuration files. The source location of the global configuration is taken from the anchor file that the configuration processor provides when invoking the configuration client.


### 2.5 Password Store

Stores security token passwords in a shared memory segment of the operating system that can be accessed by the signer. Allows security token logins to persist, until the configuration proxy is restarted, without compromising the passwords.


### 2.6 SSCD

The SSCD (Secure Signature Creation Device) is an optional hardware component that provides secure cryptographic signature creation capability to the signer.

The SSCD needs to be a PKCS \#11 (see \[[PKCS11](#Ref_PKCS11)\]) compliant hardware device that can be optionally used by the configuration proxy for signing the generated global configuration files it generates. The use of the interface requires that a PKCS \#11 compliant device driver is installed and configured in the configuration proxy system.


## 3 Interfaces


### 3.1 Downloading Configuration

Configuration clients download the generated global configuration files from the configuration proxy (protocol used for downloading configuration is described in \[[PR-GCONF](#Ref_PR-GCONF)\]). The configuration proxy uses this interface to download global configuration files from a remote configuration source.

The configuration download interface is a synchronous interface that is provided by the configuration proxy. It is used by configuration clients such as security servers and other configuration proxies.

The interface is described in more detail in \[[ARC-G](#Ref_ARC-G)\].

Should the configuration proxy fail to download the global configuration files from a remote configuration source, no updated configuration directory will be generated for that source's configuration proxy instance. The old configuration directory will remain valid until it's validity period expires.


## 4 Configuration Distribution Workflow

X-Road configuration proxy periodically downloads the global configuration from configured sources. Each global configuration consists of a number of XML files (see \[[PR-GCONF](#Ref_PR-GCONF)\] for detailed information about configuration structure). The configuration proxy then redistributes the downloaded configuration files to other configuration clients.

Configuration files are distributed in accordance with the protocol for downloading configuration (see [Section 3.1](#31-downloading-configuration)). MIME messages are generated to represent configuration directories of the global configurations being distributed.

The configuration proxy can distribute files from as many configuration sources as necessary. For each configuration source a configuration proxy instance is configured.

The following process is performed for each configuration proxy instance.

1.  A cron job sends a request to start the configuration processor for the given configuration proxy instance.

2.  Configuration processor component calls the configuration client to download the global configuration files from the configured source.

3.  Configuration processor component generates configuration directory referencing the downloaded files.

4.  Configuration processor component sends a signing request (containing hash of the directory) to the signer component. Signer signs the hash and responds with the signature.

5.  Configuration processor component updates the configuration directory to contain the received signature.

6.  Configuration processor component moves the configuration files to the distribution directory of the web server.

7.  Security servers make HTTP requests and download the configuration.

This process is illustrated in the sequence diagram in [figure 2](#Ref_Configuration_creation_sequence_diagram).


<a id="Ref_Configuration_creation_sequence_diagram" class="anchor"></a>
![](img/arc-cp_configuration_creation_sequence_diagram.png)

Figure 2. Configuration creation sequence diagram


## 5 Deployment View

The configuration proxy is deployed on a separate server on the configuration provider's side and, optionally, on the configuration client's side as well.

[Figure 3](#Ref_Configuration_proxy_deployment_example) shows a possible deployment scenario for the configuration proxy. In the presented scenario the instance \#1 central server is distributing it's internal and external configuration through a configuration proxy to instance \#1 members and an instance \#2 configuration proxy, respectively. The instance \#2 central server distributes it's internal configuration directly to instance \#2 members and it's external configuration directly to instance \#1 members.


# 參、	T-Road管理規範

## 一、	角色與權責

### (一)、	T-Road管理中心：
由國家發展委員會建置之平臺(以下簡稱本平臺)，負責規劃、推動T-Road之傳輸系統、網路、系統發展與維護及教育訓練等事項，確保T-Road傳輸通道之穩定、可靠及安全，包括：
1.	負責T-Road傳輸系統之開發設計及傳輸資料加密等安全性考量。

2.	各機關安控伺服器由管理中心審核同意後才能連上T-Road。

3.	負責彙總由安控伺服器端上傳之服務清單及授權。

4.	將最新的安控伺服器清單、服務清單及授權發布給各安控伺服器。

5.	監控T-Road整體網路、安控伺服器效能、服務可用性，並蒐集相關紀錄。

6.	定期審視作業系統漏洞修補訊息，評估作業系統變更對T-Road傳輸系統運作及安全產生之影響，並依據評估及測試結果，對系統做必要調整，並對各安控伺服器安裝機關發布作業系統更新通知。

7.	建立T-Road傳輸系統程式版本控制及安全派送機制，派送之版本應以憑證簽章，確保派送過程未經竄改。

8.	對安控伺服器之作業環境建立標準組態列表，包括作業系統版本、套件版本及相關組態設定等作為系統安全維護設定之準則。

9.	訂定T-Road資料交換網段，接受資料中心設置機關申請安控伺服器網域及核發IP位址。

10.	訂定本平臺安全傳輸協定，確保傳遞過程全程加密，並建立身分驗證確認機制，防止未經授權之機關連接本平臺。

11.	因資料傳輸效能或安全性考量，本平臺得暫時終止服務介接機關服務。
      
### (二)、	資料中心設置機關：
負責規劃、管理及提供安控伺服器，安裝本平臺提供之傳輸軟體及套件，並確保所屬機關安全連線安控伺服器，包括：
1.	遵守本平臺相關管理要求及規範。

2.	負責安控伺服器之管理及維護。

3.	配合本平臺發布之作業系統更新通知，於一週內完成系統更新。

4.	安控伺服器需安裝防毒軟體並定期更新病毒碼，對於傳送或接收之資料、附檔需進行掃描，偵測有無感染電腦病毒。

5.	當偵測到惡意程式等警訊時，應對惡意程式先行阻絕，並中止該資料之傳送，避免惡意程式蔓延至其他機關，並追查惡意程式來源，及通知本平臺、來源機關（單位）。

6.	如發生資安事件時，應儘速通知本平臺，並採取必要之因應控管措施。

### (三)、	資料提供機關：
### (四)、	資料需求機關：
### (五)、	憑證中心：

## 二、	T-Road網路環境規範 (GSN)
(一)、	資料傳輸機關應建立專屬之跨資料交換網段進行T-Road資料傳輸作業。

(二)、	機關資料傳輸如未涉及跨資料中心，應於機關內部採VPN方式進行傳輸。

(三)、	跨資料中心資料傳輸需透過資料中心設置機關介接T-Road。

(四)、	機關應考量安全性及功能需求，採專屬T-Road VPN 或T-Road Internet方式連接T-Road，並由資料中心設置機關進行電路申請，說明如下：

1.	申請專屬T-Road VPN：

	(1)	設置機關需透過機關內VPN互相傳送。

	(2)	依據機關需求申請各類型GSN VPN電路。

	(3)	T-Road內VPN IP 統一由管理中心配發。

	(4)	IP 資料中心設置單位為單位,每單位配發/24 VPN IP。

	(5)	規劃使用192.168.100.0~192.168.250.0 此範圍。

	(6)	外部IP部分依據SS數量配發，統一由 T-Road專區連出Internet。

	(7)	戶政、地政、財稅、交通監理、健保、勞保、商工等七大資料提供機關需採用專屬VPN模式，其餘資料提供機關依其資料敏感度擇適當連接方式。

2.	T-Road Internet：

	(1) 可透過既有internet電路及IP。
	
	(2)	IP需申請核可後才可連入T-Road。
	
	(3)	非資料中心設置機關需透過機關內VPN互相傳送，透過資料中心設置機關介接T-Road VPN。
	
(五)、	資料中心設置機關應依據管理中心規劃之IP及網域命名原則進行網路環境建置。

## 三、	T-Road資安事件處理規範
(一)、	T-Road管理平臺應確保資料傳輸通道高可用性及安全性，並將跨資料中心流量導流至GSN SOC中心進行異常流量監控。

(二)、	資料中心設置機關對資料中心內相關資通訊設備、軟體、服務等，其防護應達到行政院資通安全責任等級分級辦法-A級之公務機關應辦事項-技術面之要求。

(三)、	進行跨機關資料傳輸之軟、硬體設備應納入資料中心SOC監控範圍。

(四)、	機關應依據資通安全管理法及行政院頒訂之各項資訊安全規範，進行跨資料中心資料傳輸之資通安全管理。

(五)、	資料中心設置機關須建立及提供代表信箱、電話予GSN SOC，GSN SOC如監測跨資料中心資料傳輸之異常活動，將通報發生該異常活動之資料中心設置機關。

(六)、	資料中心設置機關收到GSN SOC通報後，應將初步處置情形於8小時內回覆GSN SOC。

(七)、	各資料中心SOC如監測出跨資料中心傳輸之異常流量，應同步分享情資於予GSN SOC。

(八)、	如發現跨資料中心資料傳輸資通安全事件，應依「資通安全事件通報及應變辦法」進行相關處置作業。

## 四、	T-Road身分驗證規範
(一)、	機關使用者在使用T-Road進行資料傳輸前應先在機關端進行身分認證。

(二)、	身分驗證方式包含但不限自然人憑證、New eid、健保卡、帳號密碼。

(三)、	各機關應依服務所需採取不同之身分驗證強度，身分驗證強度等級請參考附錄IAL及AAL說明。

##五、	T-Road安控伺服器規範
(一)、	機關應考量資料傳輸需求及使用量，提供安控伺服器硬體設備。

(二)、	安控伺服器應專機專用，不得安裝非必要軟體，並以防火牆及其他必要安全設施，管控與其他主機間之資料傳輸及資源存取，禁止與網際網路進行連線。

(三)、	機關需向政府憑證管理中心申請並安裝安控伺服器所需憑證。

(四)、	機關安裝安控伺服器並完成管理平臺自檢項目後始得向T-Road管理平臺進行註冊，經管理平臺核可後方能介接T-Road。

(五)、	安控伺服器之間需透過TLS憑證進行雙向驗證以及通道加密。

(六)、	機關需定期進行安控伺服器主機作業系統更新。

(七)、	機關需定期檢視安控伺服器憑證效期。

## 六、	T-Road資料傳輸規範
(一)、	T-Road資料傳輸可採用訊息傳輸及檔案傳輸方式，採用前者傳輸方式者，當需求端提出請求後，提供端須於2分鐘內產製資料並即時回應，且每次傳輸之封包大小應小於5M。

(二)、	傳輸期間發生錯誤時，安控伺服器須提供至少以下四類錯誤訊息狀態代碼：
1.	服務提供端提供錯誤回應。

2.	服務提供端無法回應。

3.	服務需求端發送的需求不符合T-Road傳輸協定。

4.	安控伺服器遇到技術錯誤。

(三)、	機關透過安控伺服器傳遞的訊息、本文皆需進行簽章。

## 七、	T-Road服務管理(API)規範 
(一)、	機關註冊至安控伺服器的服務需符合國發會訂頒之「共通性應用程式介面規範」。

(二)、	機關應透過T-Road管理平臺提供之「資料提供服務管理」以及「資料需求服務管理」功能，進行服務提供及服務取得。

(三)、	資料提供機關需將服務註冊至安控伺服器，並同步至中央控管系統。

(四)、	資料提供機關應審核資料需求機關之適法性後，才可提供需求機關介接使用；資料需求機關如有無法取得資料情事，由國發會協調之。

(五)、	服務管理(API)使用權限由資料提供機關設定之。

(六)、	資料需求機關取得之資料應依個資法要求妥善保管資料，並不得做目的外利用。

(七)、	資料提供機關得稽核資料需求機關資料保管及服務使用紀錄。

## 八、	T-Road紀錄管理規範
(一)、	安控伺服器須自動產生完整的簽章和帶時間戳記的訊息紀錄。

(二)、	機關應將跨資料中心資料傳輸之紀錄完整保存，並上傳至T-Road管理中心。

(三)、	機關應定期將安控伺服器訊息紀錄及稽核紀錄備份至外部裝置或資料庫。

(四)、	機關應定期檢視紀錄之正確性及有效性。

(五)、	跨資料中心資料傳輸紀錄應至少保留3年以備查驗。
