# go语言uniapp消息推送sdk

## 前言
针对个推与uniapp,用go语言量身打造消息推送sdk

## 文件结构
```
unipush
    |--auth         鉴权API：获取token，删除token
    |--publics      公共结构体，公共方法
    |--push         推送API
        |--single   单推送API：cid单推，别名单推，cid批量单推，别名批量单推
        |--list     批量推API：创建消息，cid批量推，别名批量推
        |--all      群推API：群推，根据条件筛选用户推送，使用标签快速推送
        |--mission  任务API：停止任务，查询定时任务，删除定时任务
    |--report       统计API（未开始）
    |--user         用户API（未开始）
    |--测试文件及其他

```

## 快速开始
```
#厂商通道普通模板+个推通道普通模板
#策略为4：先用厂商通道，失败后再用个推通道

package main

import (
	"context"
	"fmt"
	"log"
	"strconv"
	"time"

	"github.com/qnsoft/unipush/auth"
	"github.com/qnsoft/unipush/publics"
	"github.com/qnsoft/unipush/push/single"
)

func main() {
	cid := "XXXXXXXXXXXXXXXXXXXXXXXXXXXXX" // clientId
	ctxLocal := context.Background()
	tokenLocal := ""
	confLocal := publics.GeTuiConfig{
		AppId:        "", // 个推提供的id，密码等
		AppSecret:    "",
		AppKey:       "",
		MasterSecret: "",
	}
	// 1.获取token，注意这个token过期时间为一天+1秒，每分钟调用量为100次，每天最大调用量为10万次
	result, err := auth.GetToken(ctxLocal, confLocal)
	if err != nil {
		log.Fatalln("error:", err)
	}
	tokenLocal = result.Data.Token

	// 2.1ios厂商通道的参数
	iosChannel := publics.IosChannel{
		Type: "",
		Aps: &publics.Aps{
			Alert: &publics.Alert{
                Title:       "推送消息测试标题",
                Body:        "推送消息测试内容 ???",
			},
			ContentAvailable: 0,
		},
		AutoBadge:      "+1",
		PayLoad:        "",
		Multimedia:     nil,
		ApnsCollapseId: "",
	}

	// 2.2安卓通道和个推通道的普通推送参数
	notification := publics.Notification{
        Title:       "推送消息测试标题",
        Body:        "推送消息测试内容 ???",
		ClickType:   "startapp", // 打开应用首页
		BadgeAddNum: 1,
	}

	// 2.3单推需要的参数
	singleParam := single.PushSingleParam{
		RequestId: strconv.FormatInt(time.Now().UnixNano(), 10), // 请求唯一标识号
		Audience: &publics.Audience{ // 目标用户
			Cid:           []string{cid}, // cid推送数组
			Alias:         nil,           // 别名送数组
			Tag:           nil,           // 推送条件
			FastCustomTag: "",            // 使用用户标签筛选目标用户
		},
		Settings: &publics.Settings{ // 推送条件设置
			TTL: 3600000, // 默认一小时，消息离线时间设置，单位毫秒
			Strategy: &publics.Strategy{ // 厂商通道策略，具体看public_struct.go
				Default: 1,
				Ios:     4,
				St:      4,
				Hw:      4,
				Xm:      4,
				Vv:      4,
				Mz:      4,
				Op:      4,
			},
			Speed:        100, // 推送速度，设置100表示：100条/秒左右，0表示不限速
			ScheduleTime: 0,   // 定时推送时间，必须是7天内的时间，格式：毫秒时间戳
		},
		PushMessage: &publics.PushMessage{
			Duration:     "", // 手机端通知展示时间段
			Notification: &notification,
			Transmission: "",
			Revoke:       nil,
		},
		PushChannel: &publics.PushChannel{
			Ios: &iosChannel,
			Android: &publics.AndroidChannel{Ups: &publics.Ups{
				Notification: &notification,
				TransMission: "", // 透传消息内容，与notification 二选一
			}},
		},
	}

	// 3.执行单推
	singleResult, err := single.PushSingleByCid(ctxLocal, confLocal, tokenLocal, &singleParam)
	if err != nil {
		log.Fatalln("error:", err)
	}
	log.Println("result:", singleResult)
}
