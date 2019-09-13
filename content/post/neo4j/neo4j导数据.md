---
date: 2018-03-22
title: "neo4j导数据"
author: "邓子明"
tags:
    - neo4j
    - 图数据库
categories:
    - 开发经验
comment: true
---

## 1.修改配置


dbms.security.allow_csv_import_from_file_urls=true
-- load csv 命令

dbms.directories.import=import

restart neo4j

### 2.导入数据方法1

load csv with headers from "file:///path/to/file" as row
create (:Employee {employeeId:toInt(row.id),first_name:row,first_name,title:row.title });

### 3. 导入数据方法2

dbms.directories.import=/var/lib/neo4j/import/

注释：
# dbms.security.allow_csv_import_from_file_urls=true
将文件放入 ：/var/lib/neo4j/import/  文件夹下，直接输入文件名即可：

load csv with headers from "file:///filename" as row
create (:Employee {employeeId:toInt(row.id),first_name:row,first_name,title:row.title });

### 4.初始化导数据

新建每个节点和关系的header文件和数据文件：

```bash
# vertex header
phone:ID(PHONE),isblack,ismedia,iscuishou
# edge header
:START_ID(USERID),:END_ID(PHONE)
```

然后导数据：
```bash
neo4j-import \
 --into /data/neo4j/graph/all20180417.db \
 --skip-duplicate-nodes true \
 --skip-bad-relationships true \
 --ignore-extra-columns true \
 --ignore-empty-strings true \
 --bad-tolerance 10000000 \
  --processors 56 \
 --id-type string \
 --max-memory 170G \
--nodes:LBS "/data1/neo4j/data/offline/graph_kg/header/kg_v2_v_lbs.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_v_lbs.txt"  \
--nodes:IDCARD "/data1/neo4j/data/offline/graph_kg/header/kg_v2_v_idcard.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_v_idcard.txt"  \
--nodes:GNHID "/data1/neo4j/data/offline/graph_kg/header/kg_v2_v_gnhid.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_v_gnhid.txt"  \
--nodes:QQ "/data1/neo4j/data/offline/graph_kg/header/kg_v2_v_qq.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_v_qq.txt"  \
--nodes:WEIXIN "/data1/neo4j/data/offline/graph_kg/header/kg_v2_v_weixin.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_v_weixin.txt"  \
--nodes:EMAIL "/data1/neo4j/data/offline/graph_kg/header/kg_v2_v_email.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_v_email.txt"  \
--nodes:DEVICETOKEN "/data1/neo4j/data/offline/graph_kg/header/kg_v2_v_devicetoken.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_v_devicetoken.txt"  \
--nodes:COMPANY "/data1/neo4j/data/offline/graph_kg/header/kg_v2_v_company.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_v_company.txt"  \
--nodes:IP "/data1/neo4j/data/offline/graph_kg/header/kg_v2_v_ip.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_v_ip.txt"  \
--nodes:ORDERID "/data1/neo4j/data/offline/graph_kg/header/kg_v2_v_orderid.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_v_orderid.txt"  \
--nodes:USERID "/data1/neo4j/data/offline/graph_kg/header/kg_v2_v_userid.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_v_userid.txt"  \
--nodes:PHONE "/data1/neo4j/data/offline/graph_kg/header/kg_v2_v_phone.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_v_phone.txt"  \
--nodes:WIFI "/data1/neo4j/data/offline/graph_kg/header/kg_v2_v_wifi.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_v_wifi.txt"  \
--relationships:USERID_PHONE_EMG "/data1/neo4j/data/offline/graph_kg/header/kg_v3_e_userid_phone_emg.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v3_e_userid_phone_emg.txt"  \
--relationships:USERID_PHONE_LOAN "/data1/neo4j/data/offline/graph_kg/header/kg_v3_e_userid_phone_loan.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v3_e_userid_phone_loan.txt"  \
--relationships:USERID_COMPANY_LOAN "/data1/neo4j/data/offline/graph_kg/header/kg_v3_e_userid_company_loan.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v3_e_userid_company_loan.txt"  \
--relationships:USERID_LBS_LOAN "/data1/neo4j/data/offline/graph_kg/header/kg_v3_e_userid_lbs_loan.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v3_e_userid_lbs_loan.txt"  \
--relationships:USERID_DEVICETOKEN_LOAN "/data1/neo4j/data/offline/graph_kg/header/kg_v3_e_userid_devicetoken_loan.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v3_e_userid_devicetoken_loan.txt"  \
--relationships:USERID_LBS "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_lbs.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_lbs.txt"  \
--relationships:USERID_COMPANY_GJJ "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_company_gjj.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_company_gjj.txt"  \
--relationships:USERID_IDCARD "/data1/neo4j/data/offline/graph_kg/header/kg_v3_e_userid_idcard.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v3_e_userid_idcard.txt"  \
--relationships:USERID_COMPANY_GNH_USERS "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_company_gnh_users.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_company_gnh_users.txt"  \
--relationships:USERID_GNHID_USERS "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_gnhid_users.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_gnhid_users.txt"  \
--relationships:USERID_IP_GNH_USERS "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_ip_gnh_users.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_ip_gnh_users.txt"  \
--relationships:USERID_PHONE_GNH_EMG_USERS "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_phone_gnh_emg_users.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_phone_gnh_emg_users.txt"  \
--relationships:USERID_QQ_GNH_USERS "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_qq_gnh_users.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_qq_gnh_users.txt"  \
--relationships:ORDERID_IP_GNH_USERS "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_orderid_ip_gnh_users.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_orderid_ip_gnh_users.txt"  \
--relationships:USERID_ORDERID "/data1/neo4j/data/offline/graph_kg/header/kg_v3_e_userid_orderid.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v3_e_userid_orderid.txt" \
--relationships:USERID_DEVICETOKEN_RISKBRAIN_UMID "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_devicetoken_riskbrain_umid.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_devicetoken_riskbrain_umid.txt" \
--relationships:USERID_DEVICETOKEN_USER_DEVICES "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_devicetoken_t_user_devices.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_devicetoken_t_user_devices.txt" \
--relationships:USERID_DEVICETOKEN_USER_EVENT "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_devicetoken_user_event.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_devicetoken_user_event.txt" \
--relationships:USERID_COMPANY_USER_INFO_EXT "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_company_user_info_ext.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_company_user_info_ext.txt" \
--relationships:USERID_IP_CUSTINFO "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_ip_custinfo.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_ip_custinfo.txt" \
--relationships:USERID_IP_USER_EVENT "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_ip_user_event.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_ip_user_event.txt" \
--relationships:USERID_EMAIL_USER_EVENT "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_email_user_event.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_email_user_event.txt" \
--relationships:USERID_EMAIL_EXTMAIL_USER_MAILACCOUNT "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_email_user_mailaccount.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_email_user_mailaccount.txt" \
--relationships:USERID_QQ_USER_EVENT "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_qq_user_event.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_qq_user_event.txt" \
--relationships:USERID_WEIXIN_USER_EVENT "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_weixin_user_event.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_weixin_user_event.txt" \
--relationships:USERID_PHONE_USER_EVENT "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_phone_user_event.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_phone_user_event.txt" \
--relationships:USERID_PHONE_CUSTINFO "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_phone_custinfo.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_phone_custinfo.txt" \
--relationships:USERID_PHONE_USERCENTER "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_phone_usercenter.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_phone_usercenter.txt" \
--relationships:USERID_PHONE_USER_EMG_CONTACT "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_phone_user_emg_contact.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_phone_user_emg_contact.txt" \
--relationships:USERID_PHONE_TEL "/data1/neo4j/data/offline/graph_kg/header/kg_v3_e_userid_phone_tel.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v3_e_userid_phone_tel.txt" \
--relationships:USERID_PHONE_CONTACT "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_phone_contact.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_phone_contact.txt" \
--relationships:PHONE_PHONE_TEL "/data1/neo4j/data/offline/graph_kg/header/kg_v3_e_phone_phone_tel.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v3_e_phone_phone_tel.txt" \
--relationships:PHONE_PHONE_CONTACT "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_phone_phone_contact.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_phone_phone_contact.txt" \
--relationships:USERID_WIFI "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_userid_wifi.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_userid_wifi.txt" \
--relationships:ORDERID_WIFI "/data1/neo4j/data/offline/graph_kg/header/kg_v2_e_orderid_wifi.txt,/data1/neo4j/data/offline/graph_kg/data/kg_v2_e_orderid_wifi.txt" \
          > /data/neo4j/graph/all20180417.log 2>&1 &
```
