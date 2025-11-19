# IOD APIs and Customer Interactions

> Confluence Page ID: 675856842772, Version: 8

Information contained in these pages is accurate to the best of our ability. If something needs to be clarified or corrected, please reach out.

Table of Contents

Overview
========

This document captures current IOD APIs and how they are used to order services and gaps. It also talks short term fix for pressing customer issues with API like customers are having

* Getting status of order via webhook
* Issue setting up sandbox without functional webhook for sandbox environment.

Details
=======

### Current API and Purpose

* Provide information what the current APIs are available and their purpose and gaps.

![IOD External API Current State.png](images/675856842772-0-IOD%20External%20API%20Current%20State.png)

### Option# 1 Current APIs with API to get order request status reporting orderId and serviceId

* use id (b2b) which returned today in orderRequest POST as reference to collect status/state and orderId and serviceId.  
  {  
   **"id": "57ab4320-9aaf-11ee-8cbc-bf8b3daa320c",**  
   "externalId": 1702578835,  
   ....  
  }
* Api will return following information to client. it can follow current data format to be compatible with POST payload which is extension TMF payload design.

  + **state/status** RUNNING, SUCCEEDED , FAILED , for status follow [API Standard: Long-Runing Operations (LRO)](/wiki/spaces/NAPA/pages/675812048943/API+Standard+Long-Runing+Operations+LRO#3.5-State-Machine)

    - Status of the order Request
  + **orderId** (88 number, once we have it available in backend or when orderRequest goes to SUCCEEDED)

    - The order id which is need to submit service-configuration request using serviceProvisioning API
  + **serviceId** (77 number, once we have it available in backend or when orderRequest goes to SUCCEEDED)

    - The service id which is needed to get the information about service using serviceProvisioning API
* Use Retry After header attributes to guide client on configurable delay, follow [API Standard: Long-Runing Operations (LRO)](/wiki/spaces/NAPA/pages/675812048943/API+Standard+Long-Runing+Operations+LRO#3.6-Client-Polling-Strategy)

![IOD External API - Get well quick fix 11-10-2025.png](images/675856842772-1-IOD%20External%20API%20-%20Get%20well%20quick%20fix%2011-10-2025.png)

### Option#2 Current APIs with enhancement to how customer submit order and API they use

* Provide information what can be done in short term/ minimal effort to improve Aysnc experience of order submission API
* Remove dependency on webhook to get status.
* New URI for order submission ( compatible with current orderRequest URI for input and response) and get order status , collect serviceId etc and start moving in direction suggested by API style guide for future enhancement.

![IOD External API - Get well quick fix 11-06-2025.png](images/675856842772-2-IOD%20External%20API%20-%20Get%20well%20quick%20fix%2011-06-2025.png)

### Future Sate of IoD APIs

* It is expected that IoD will be ordered as Fabric Connection using North Star Design
* API URIs would look different and payload will be different as data dependency will be changed.
* Future IOD API state is not expected to be define here. TODO: add reference to documentation when ready.

Decision
========

Option#1 is selected as proposal with understanding that it will require minimal development effort without compromising much on the current API experience. UI and naasOpsDashboard already provide this kinda visibility so it is expected that it won’t be difficult to create the relationship as API data will be there or collected from backend systems. This will solve the immediate issues customer are facing. With understanding that IoD APIs needs overhaul to match target resource model inline with API Strategy initiative.

Appendix
========

<https://developer.lumen.com/apis/internet-on-demand>