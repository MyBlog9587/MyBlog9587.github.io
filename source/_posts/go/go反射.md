---
title: go反射
date: 2023-06-09 17:16:35
categories:
- go
tags:
- go
- reflect
---


# 将结构体的成员解析为map

```go
package main

import (
	"fmt"
	"reflect"
)

type Person struct {
	Name   string
	Age    int
	Gender string
	Sir    *Sir
}

type Sir struct {
	Name string
	Id   uint32
}

func check(condition map[string]interface{}) error {
	for k, v := range condition {
		switch reflect.TypeOf(v).Kind() {
		case reflect.Int:
			fmt.Printf("key: %s\tvalue type %T:%v\n", k, v, v)
		case reflect.Uint32:
			fmt.Printf("key: %s\tvalue type %T:%v\n", k, v, v)
		case reflect.Struct:
			fmt.Printf("key: %s\tvalue type %T:%v\n", k, v, v)
		case reflect.String:
			fmt.Printf("key: %s\tvalue type %T:%v\n", k, v, v)
		default:
			fmt.Printf("other key: %s\tvalue type %T:%v\n", k, v, v)
		}
	}
	return nil
}

func main() {
	sir := &Sir{
		Name: "sir",
		Id:   1,
	}

	p := Person{
		Name:   "Alice",
		Age:    30,
		Gender: "Female",
		Sir:    sir,
	}

	if m, err := structToMap(p); err != nil {
		fmt.Println(err)
	} else {
        // map: map[Age:30 Gender:Female Name:Alice Sir:map[Id:1 Name:sir]]
		fmt.Printf("map: %+v\n", m)
        // m.sir type: map[string]interface {}     value: map[Id:1 Name:sir]
		fmt.Printf("m.sir type: %T\tvalue: %+v\n", m["Sir"], m["Sir"])

        // key: Name       value type string:Alice
        // key: Age        value type int:30
        // key: Gender     value type string:Female
        // default key: Sir        value type map[string]interface {}:map[Id:1 Name:sir]
		check(m)
	}
}





func structToMap(obj interface{}) (map[string]interface{}, error) {
	result := make(map[string]interface{})
	v := reflect.ValueOf(obj)
	if v.Kind() != reflect.Struct {
		return result, fmt.Errorf("structToMap only accepts structs; got %T", v)
	}

	t := v.Type()

	for i := 0; i < v.NumField(); i++ {
		field := t.Field(i)
		value := v.Field(i)

		if value.CanInterface() {
			if value.Kind() == reflect.Ptr {
				value = value.Elem()
			}

			filedName := field.Name
			if value.Kind() == reflect.Struct {
				subMap, err := structToMap(value.Interface())
				if err != nil {
					return nil, err
				}
				result[filedName] = subMap
			} else {
				result[filedName] = value.Interface()
			}
		}
	}

	return result, nil
}

```