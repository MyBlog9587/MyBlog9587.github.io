---
title: gorm库使用异常总结
date: 2023-06-13 11:58:49
categories: [go]
tags:
- go
- gorm
---

## demo 代码如下：

```go 	
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"gorm.io/driver/postgres"
	"gorm.io/gorm"
	"gorm.io/gorm/schema"
)

type PolicyInfo struct {
	Id           uint32 `gorm:"column:id;primaryKey" mapstructure:"id"`            // 规则id
	RuleName     string `gorm:"column:rule_name" mapstructure:"rule_name"`         // 规则名称
	Action       uint32 `gorm:"column:action" mapstructure:"action"`               // 处置动作类型
	RedirectIp   string `gorm:"column:redirect_ip" mapstructure:"redirect_ip"`     // 重定向ip
	RulePriority uint32 `gorm:"column:rule_priority" mapstructure:"rule_priority"` // 规则优先级
	Status       uint32 `gorm:"column:status" mapstructure:"status"`               // 规则状态

	Eid       string `gorm:"column:eid" mapstructure:"eid"`               // 企业id
	IsDeleted bool   `gorm:"column:is_deleted" mapstructure:"is_deleted"` // 是否删除

	FormJson string `gorm:"column:form_json" mapstructure:"form_json"` // 表单json

	CreateTime    string `gorm:"column:create_time" mapstructure:"create_time"`         // 创建时间
	UpdateTime    string `gorm:"column:update_time" mapstructure:"update_time"`         // 更新时间
	OpenCloseTime string `gorm:"column:open_close_time" mapstructure:"open_close_time"` // 开关时间

	Severity    []uint32    `gorm:"column:severity" json:"severity,omitempty"`
	Blacklist   []string    `gorm:"column:blacklist" json:"blacklist,omitempty"`
	DefaultType uint32      `gorm:"column:default_type" json:"default_type,omitempty"`
	DomainType  []uint32    `gorm:"column:domain_type" json:"domain_type,omitempty"`
	EffectArea  *EffectArea `json:"effect_area,omitempty"` // 生效范围
	EffectCount uint32      `json:"effect_count,omitempty"`
}

type RequestRuleList struct {
	Action       uint32      `protobuf:"varint,1,opt,name=action,proto3" json:"action,omitempty"`                                  // 处置动作类型
	DomainType   []uint32    `protobuf:"varint,2,rep,packed,name=domain_type,json=domainType,proto3" json:"domain_type,omitempty"` // 域名类型
	Blacklist    []string    `protobuf:"bytes,3,rep,name=blacklist,proto3" json:"blacklist,omitempty"`                             // 黑名单
	EffectArea   *EffectArea `protobuf:"bytes,4,opt,name=effect_area,json=effectArea,proto3" json:"effect_area,omitempty"`         // 生效范围
	RedirectIp   string      `protobuf:"bytes,5,opt,name=redirect_ip,json=redirectIp,proto3" json:"redirect_ip,omitempty"`         // 重定向ip
	RuleName     string      `protobuf:"bytes,6,opt,name=rule_name,json=ruleName,proto3" json:"rule_name,omitempty"`               // 规则名称
	RulePriority uint32      `protobuf:"varint,7,opt,name=rule_priority,json=rulePriority,proto3" json:"rule_priority,omitempty"`  // 规则优先级
	Severity     []uint32    `protobuf:"varint,8,rep,packed,name=severity,proto3" json:"severity,omitempty"`                       // 威胁等级
}

type FormJson struct {
	DefaultType    uint32   `json:"default_type,omitempty"`     // 默认类型
	DomainType     []uint32 `json:"domain_type,omitempty"`      // 域名类型
	Blacklist      []string `json:"blacklist,omitempty"`        // 黑名单
	Severity       []uint32 `json:"severity,omitempty"`         // 威胁等级
	DataSourceUids []string `json:"data_source_uids,omitempty"` // 情报
	GroupCheckAll  uint32   `json:"group_check_all,omitempty"`  // 安全域
	Version        string   `json:"version,omitempty"`
}

type EffectArea struct {
	GroupCheckAll  uint32   `json:"group_check_all"`
	DataSourceUids []string `json:"data_source_uids"`
}

// 切片去重
func removeDuplicates(s []string) []string {
	seen := make(map[string]bool)
	result := []string{}
	for _, v := range s {
		if _, ok := seen[v]; !ok {
			seen[v] = true
			result = append(result, v)
		}
	}
	return result
}

const (
	postgre_data_source = "host=10.229.3.59 user=light password=HAbjC6RByLUP5qpSUhtAFWtPfmw0BPvzqJNCYgvu65bJVSVomX2gJiRlvxspTQpZ dbname=light search_path=light port=31387 sslmode=disable TimeZone=Asia/Shanghai"
	MaxIdeConn          = 10
	MaxOpenConn         = 100
	ConnMaxLife         = 60
)

func GetDB() (*gorm.DB, error) {
	db, err := gorm.Open(postgres.New(postgres.Config{
		DSN:                  postgre_data_source,
		PreferSimpleProtocol: true, // disables implicit prepared statement usage
	}), &gorm.Config{
		PrepareStmt: true,
		NamingStrategy: schema.NamingStrategy{
			SingularTable: true,
		},
	})

	if err != nil {
		return nil, fmt.Errorf("Unable to connection to database! dsn:%v err:%s", postgre_data_source, err)
	}

	sqlDb, err := db.DB()
	if err != nil {
		fmt.Errorf("get db is err:%s", err)
		return nil, err
	}

	// defer sqlDb.Close()

	// 构建连接池
	sqlDb.SetMaxIdleConns(int(MaxIdeConn))
	sqlDb.SetMaxOpenConns(int(MaxOpenConn))
	sqlDb.SetConnMaxLifetime(ConnMaxLife)

	if err = sqlDb.Ping(); err != nil {
		fmt.Errorf("models.Ping err: %v", err)
		return nil, err
	}
	return db, nil
}

func main() {

	ctx, cancle := context.WithCancel(context.Background())
	defer cancle()

	eid := "C6ADCD26-E63B-11EC-BC53-B62BBA637A35"

	effectArea := &EffectArea{
		GroupCheckAll:  0,
		DataSourceUids: []string{"sdsd", "test0914"},
	}

	in := RequestRuleList{
		RuleName:     "test",
		Action:       222,
		Blacklist:    []string{"afdasfsa", "test0914"},
		DomainType:   []uint32{10100, 10017, 10021, 10025, 10035, 10037, 10041, 10042, 10043, 10011, 10012, 10013, 10014, 10015},
		EffectArea:   effectArea,
		RedirectIp:   "1.1.1.1",
		RulePriority: 126,
		Severity:     []uint32{1, 3, 5},
	}
	// 构造数据
	black_list := removeDuplicates(in.Blacklist)

	form_json := FormJson{
		Blacklist:      black_list,
		DomainType:     in.DomainType,
		Severity:       in.Severity,
		GroupCheckAll:  in.EffectArea.GroupCheckAll,
		DataSourceUids: in.EffectArea.DataSourceUids,
		Version:        "1.0.0",
	}

	jsonBytes, err := json.Marshal(form_json)
	if err != nil {
		fmt.Println(err)
		return
	}
	jsonStr := string(jsonBytes)
	fmt.Println(jsonStr)

	now := time.Now().Format("2006-01-02 15:04:05")

	rule_info := PolicyInfo{
		Action:       in.Action,
		Status:       1,
		RuleName:     in.RuleName,
		RulePriority: in.RulePriority,

		IsDeleted:     false,
		RedirectIp:    in.RedirectIp,
		CreateTime:    now,
		UpdateTime:    now,
		OpenCloseTime: now,
		Eid:           eid,

		FormJson: jsonStr,
	}

	db, err := GetDB()
	if err != nil {
		fmt.Errorf("get db is err:%s", err)
		return
	}

	result := db.Table("s_rule").WithContext(ctx).Create(&rule_info)
	if result.Error != nil && result.Error != gorm.ErrRecordNotFound {
		fmt.Println(result.Error)
		return
	}
    
	fmt.Printf("result.Id: %d\tinsert rule row: %d\n", rule_info.Id, result.RowsAffected)
}
```

运行异常：
```bash
# [14.697ms] [rows:0] INSERT INTO "s_rule" ("rule_name","action","redirect_ip","rule_priority","status","eid","is_deleted","form_json","create_time","update_time","open_close_time","severity","blacklist","default_type","domain_type","effect_count") VALUES ('test',222,'1.1.1.1',126,1,'C6ADCD26-E63B-11EC-BC53-B62BBA637A35',false,'{"domain_type":[10100,10017,10021,10025,10035,10037,10041,10042,10043,10011,10012,10013,10014,10015],"blacklist":["afdasfsa","test0914"],"severity":[1,3,5],"data_source_uids":["sdsd","test0914"],"version":"1.0.0"}','2023-06-13 12:10:14','2023-06-13 12:10:14','2023-06-13 12:10:14',(NULL),(NULL),0,(NULL),0) RETURNING "id"
ERROR: column "severity" of relation "s_rule" does not exist (SQLSTATE 42703)
```

是因为将 FormJson 内容进行了展开，并追加到了 sql 语句中，导致了异常。

## 修复方法1:
问题出现的原因是 gorm 标签的使用不正确。在结构体 PolicyInfo 中，FormJson 字段的 gorm 标签声明在了错误的位置，导致其后面的字段都被认为是 FormJson 的子字段。

要解决这个问题，需要将 gorm 标签放置在结构体字段定义的后面，并且确保标签的格式正确。以下是修复后的代码示例：
```go
type PolicyInfo struct {
	Id           uint32   `gorm:"column:id;primaryKey" mapstructure:"id"` // 规则id
	RuleName     string   `gorm:"column:rule_name" mapstructure:"rule_name"` // 规则名称
	Action       uint32   `gorm:"column:action" mapstructure:"action"` // 处置动作类型
	RedirectIp   string   `gorm:"column:redirect_ip" mapstructure:"redirect_ip"` // 重定向ip
	RulePriority uint32   `gorm:"column:rule_priority" mapstructure:"rule_priority"` // 规则优先级
	Status       uint32   `gorm:"column:status" mapstructure:"status"` // 规则状态

	Eid          string   `gorm:"column:eid" mapstructure:"eid"` // 企业id
	IsDeleted    bool     `gorm:"column:is_deleted" mapstructure:"is_deleted"` // 是否删除
	FormJson     string   `gorm:"column:form_json" mapstructure:"form_json"` // 表单json
	CreateTime   string   `gorm:"column:create_time" mapstructure:"create_time"` // 创建时间
	UpdateTime   string   `gorm:"column:update_time" mapstructure:"update_time"` // 更新时间
	OpenCloseTime string   `gorm:"column:open_close_time" mapstructure:"open_close_time"` // 开关时间
	Severity     []uint32 `gorm:"-" json:"severity,omitempty"` // 临时排除
	Blacklist    []string `gorm:"-" json:"blacklist,omitempty"` // 临时排除
	DefaultType  uint32   `gorm:"-" json:"default_type,omitempty"` // 临时排除
	DomainType   []uint32 `gorm:"-" json:"domain_type,omitempty"` // 临时排除
	EffectArea   *EffectArea `json:"effect_area,omitempty"` // 生效范围
	EffectCount  uint32   `gorm:"-" json:"effect_count,omitempty"` // 临时排除
}
```



## 修复方法2:
在创建记录时，GORM 会自动将结构体中的所有字段都写入 SQL 中。要达到你想要的效果（只插入选定的字段），你可以使用 Select 方法指定要插入的字段
通过使用 Select 方法，在 SQL 中插入的字段将只包括所指定的字段，而不会在 open_close_time 后面追加其他字段。
```go
	result := p.db.Table(biz.PolicyTableName).
		WithContext(ctx).
		Select("RuleName", "Action", "RedirectIp", "RulePriority", "Status", "Eid", "IsDeleted", "FormJson", "CreateTime", "UpdateTime", "OpenCloseTime").
		Create(&rule_info)
```
