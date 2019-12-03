---
title: netease-cloud-music
date: 2019-12-03 17:10:50
tags:
	- 爬虫
---
爬取网易云音乐：
https://blog.csdn.net/qq_36457148/article/details/80872045
https://www.jianshu.com/p/069e88181488
https://blog.csdn.net/xiaoming_xiaoli/article/details/88019016

*****

幂模运算中的幂指数：
`b = '010001'`
幂模运算中的模数：
`c = '00e0b509f6259df8642dbc35662901477df22677ec152b5ff68ace615bb7b725152b3ab17a876aea8a5aa76d2e417629ec4ee341f56135fccf695280104e0312ecbda92557c93870114af6c9d05c4f7f0c3685b7a46bee255932575cce10b424d813cfe4875d3e82047b97ddef52741d546b8e289dc6935b3ece0462db0a22b8e7'`
AES 加密 KEY：
`d = '0CoJUm6Qyw8W8jud'`
AES 加密向量：
`iv = '0102030405060708'`
*****
**1.keyword --> musicId，根据搜索关键字获取音乐编号**
request(application/x-www-form-urlencoded):
`https://music.163.com/weapi/cloudsearch/get/web?csrf_token=`
参数：
`params=PARAMS&encSecKey=ENC_SEC_KEY`
`content = '{hlpretag:"<span class=s-fc7>",hlposttag:"</span>",s:"'+keyword+'",type:"1",offset:"'+offset+'",total:"true",limit:"'+limit+'",csrf_token:""}'`
AES 加密得到 PARAMS（其中 rand_code 为16位随机字符串）：
`PARAMS = base64.encode(AES.encrypt(base64.encode(AES.encrypt(content, d, iv)), rand_code, iv))`
java 实现：
```java
aesEncrypt(aesEncrypt(content, d, iv), rand_code, iv)
aesEncrypt(content, key, iv):
Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
byte[] raw = key.getBytes();
SecretKeySpec keySpec = new SecretKeySpec(raw, "AES");
// 使用 CBC 模式，需要一个向量 ivSpec，可增加加密算法的强度
IvParameterSpec ivSpec = new IvParameterSpec(iv.getBytes());
cipher.init(Cipher.ENCRYPT_MODE, keySpec, ivSpec);
byte[] encrypted = cipher.doFinal(content.getBytes(StandardCharsets.UTF_8));
// 使用 BASE64 做转码
return Base64.getEncoder().encodeToString(encrypted);
```
幂模运算得到 ENC_SEC_KEY（其中 rand_code 为16位随机字符串，要与 AES 加密中的相同）：
`ENC_SEC_KEY = format(int(rand_code, 16) ** int(b, 16) % int(c, 16),'x').zfill(256)`
java 实现：
```java
StringBuilder reverse = new StringBuilder(randCode).reverse();
BigInteger base = new BigInteger(StringUtil.byteArrayToHex(reverse.toString().getBytes()), 16);
// b 为 010001
BigInteger exponent = new BigInteger(b, 16);
// c 为 00e0b509f6259df8642dbc35662901477df22677ec152b5ff68ace615bb7b725152b3ab17a876aea8a5aa76d2e417629ec4ee341f56135fccf695280104e0312ecbda92557c93870114af6c9d05c4f7f0c3685b7a46bee255932575cce10b424d813cfe4875d3e82047b97ddef52741d546b8e289dc6935b3ece0462db0a22b8e7
BigInteger modulus = new BigInteger(c, 16);
String result = base.modPow(exponent, modulus).toString(16);
int length = result.length();
if (length > 256) {
    return result.substring(0, 256);
}
return String.join("", Collections.nCopies((256 - length), "0")) + result;
```
response:
```java
{
    "result": {
        "songs": [
            {
                "name": "好久不见",
                "id": 65538,
                "pst": 0,
                "t": 0,
                "ar": [
                    {
                        "id": 2116,
                        "name": "陈奕迅",
                        "tns": [],
                        "alias": []
                    }
                ],
                "alia": [],
                "pop": 100,
                "st": 0,
                "rt": "600902000005371150",
                "fee": 8,
                "v": 74,
                "crbt": null,
                "cf": "",
                "al": {
                    "id": 6434,
                    "name": "认了吧",
                    "picUrl": "http://p1.music.126.net/o_OjL_NZNoeog9fIjBXAyw==/18782957139233959.jpg",
                    "tns": [],
                    "pic_str": "18782957139233959",
                    "pic": 18782957139233959
                },
                "dt": 250000,
                "h": {
                    "br": 320000,
                    "fid": 0,
                    "size": 10022704,
                    "vd": 614
                },
                "m": {
                    "br": 192000,
                    "fid": 0,
                    "size": 6013639,
                    "vd": 2511
                },
                "l": {
                    "br": 128000,
                    "fid": 0,
                    "size": 4009107,
                    "vd": 2095
                },
                "a": null,
                "cd": "1",
                "no": 7,
                "rtUrl": null,
                "ftype": 0,
                "rtUrls": [],
                "djId": 0,
                "copyright": 1,
                "s_id": 0,
                "mark": 8192,
                "cp": 7003,
                "mv": 281152,
                "mst": 9,
                "rtype": 0,
                "rurl": null,
                "publishTime": 1177344000000,
                "privilege": {
                    "id": 65538,
                    "fee": 8,
                    "payed": 0,
                    "st": 0,
                    "pl": 128000,
                    "dl": 0,
                    "sp": 7,
                    "cp": 1,
                    "subp": 1,
                    "cs": false,
                    "maxbr": 999000,
                    "fl": 128000,
                    "toast": false,
                    "flag": 4
                }
            }
        ],
        "songCount": 429
    },
    "code": 200
}
```
*****
**2.musicId --> musicUrl，根据音乐编号获取音乐播放/下载地址**
request(application/x-www-form-urlencoded):
`https://music.163.com/weapi/song/enhance/player/url/v1?csrf_token=`
参数：
`params=PARAMS&encSecKey=ENC_SEC_KEY`
`content = '{"ids":"[MUSIC_ID]","level":"standard","encodeType":"aac","csrf_token":""}'`
response:
```java
{
    "data": [
        {
            "id": 65538,
            "url": "http://m10.music.126.net/20191106150935/9d969a9ca15ff95b352f397ec2e8ec8b/ymusic/d8ce/08c3/986c/844405c5672efe9b10076bab25d7bce2.mp3",
            "br": 128000,
            "size": 4009107,
            "md5": "844405c5672efe9b10076bab25d7bce2",
            "code": 200,
            "expi": 1200,
            "type": "mp3",
            "gain": 0,
            "fee": 8,
            "uf": null,
            "payed": 0,
            "flag": 4,
            "canExtend": false,
            "freeTrialInfo": null,
            "level": "standard",
            "encodeType": "mp3"
        }
    ],
    "code": 200
}
```
