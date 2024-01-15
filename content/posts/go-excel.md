---
title: Go实现Excel文件导出
date: 2023-08-24 17:18:01
tags: 
- "Excel"
---

## 背景
在日常开发中，我们经常会遇到需要导出Excel文件的场景，在使用Go开发时有一个比较流行的库[excelize](https://github.com/qax-os/excelize)，但支持的功能比较有限。

为了业务中更易用，于是想到可以基于go tag实现结构化的Excel导出


## 详细设计
常规的，在Excel中我们只关注两部分，表头和内容。

### 表头

针对表头的管理，设计了下面的结构记录表头的元数据

```go
type Meta struct {
    // 序号
    Idx int `json:"idx"`
    // 字段名称
    Key string `json:"key"`
    // excel列的标识 A B
    Col string `json:"col"`
    // 表头名称
    Value string `json:"value"`
}
```

struct天然的就提供这个能力，我们可以通过添加自定义tag的方式获取到表头名称，通过反射直接获取

### 内容

针对内容的管理，我们不需要很复杂的展示逻辑，仅支持字符串和数字两种基础类型即可，这样就极大的简化了我们的操作，我们只需要提供一个这样的结构

```go
type EasyExport struct {
	ID    string `excel:"id"`
	Name  string `excel:"名称"`
	Name2 string `excel:"名称2"`
	Num   int64  `excel:"数量"`
}
```

在我们实现导出逻辑时，只需要关注如果把数据转换成这个结构即可，搭配oss文件存储，只需如下的代码，我们就可以很便利的完成需求

```go
    func doExport(ctx context.Context, ids []int64) error {
        // 查询出要导出文件的结构化数据
        dataExports, err := queryDataExports(ctx, ids)
        if err != nil {
            return err
        }
        // 获取到数据流buffer
        buffer, err := excel.NewExportClient().Export(ctx, "Sheet1", dataExports)
        if err != nil {
            return err
        }
        // 自定义的oss客户端
        err = oss.GetClient().UploadXlsx(ctx, "xxxx.xlsx", buffer)
        if err != nil {
            return err
        }
        return nil
    }
```

#### 详细实现

```go
package excel

import (
	"bytes"
	"context"
	"errors"
	"fmt"
	"reflect"
	"strings"

	excelize "github.com/xuri/excelize/v2"
)

var (
	supportKinds map[reflect.Kind]struct{} = map[reflect.Kind]struct{}{
		reflect.Slice: {},
		reflect.Array: {},
	}
	excelClient      *DefaultExcelClient
)

func NewExportClient() *DefaultExcelClient {
	excelClient = &DefaultExcelClient{}
	return excelClient
}

type DefaultExcelClient struct {
}

type Meta struct {
	// 序号
	Idx int `json:"idx"`
	// 字段名称
	Key string `json:"key"`
	// excel列的标识 A B
	Col string `json:"col"`
	// 表头名称
	Value string `json:"value"`
}

type ExportTask struct {
	sheetName string
	metas     map[string]*Meta
	f         *excelize.File
}

func NewExportTask(f *excelize.File, sheetName string, metas map[string]*Meta) *ExportTask {
	return &ExportTask{
		sheetName: sheetName,
		metas:     metas,
		f:         f,
	}
}

func (d *DefaultExcelClient) Export(ctx context.Context, sheetName string, obj interface{}, config *ExportOption) (buffer *bytes.Buffer, err error) {
	f := excelize.NewFile()
	defer func() {
		err = f.Close()
		if err != nil {
			// add log
		}
	}()
	// 创建一个工作表
	sheet, err := f.NewSheet(sheetName)
	if err != nil {
		return nil, err
	}
	metas, err := d.parseMetaMap(obj)
	if err != nil {
		return nil, err
	}
	task := NewExportTask(f, sheetName, metas)
	err = task.WriteObj(obj)
	if err != nil {
		return nil, err
	}
	f.SetActiveSheet(sheet)
	return f.WriteToBuffer()
}

func (d *DefaultExcelClient) parseMetaMap(obj interface{}) (map[string]*Meta, error) {
	refValue := reflect.TypeOf(obj)
	kind := refValue.Kind()
	if _, ok := supportKinds[kind]; !ok {
		return nil, fmt.Errorf("kind not support. %v", kind)
	}
	elem := refValue.Elem()
	if elem.Kind() == reflect.Ptr {
		// 如果是指针类型，获取指针指向的struct
		elem = elem.Elem()
	}
	if elem.Kind() != reflect.Struct {
		return nil, errors.New("kind not support")
	}
	result := make(map[string]*Meta, 0)
	for i := 0; i < elem.NumField(); i++ {
		col, err := excelize.ColumnNumberToName(i + 1)
		if err != nil {
			return nil, err
		}
		field := elem.Field(i)
		tagValue := strings.Split(field.Tag.Get("excel"), ",")
		value := field.Name
		if len(tagValue) >= 1 {
			value = tagValue[0]
		}
		meta := &Meta{
			Idx:   i,
			Key:   field.Name,
			Col:   col,
			Value: value,
		}
		result[meta.Key] = meta
	}
	return result, nil
}

func (s *ExportTask) WriteObj(obj interface{}) error {
	metaMap := s.metas
	name := s.sheetName
	f := s.f
	for _, meta := range metaMap {
		// 写excel表头
		col := fmt.Sprintf("%s%d", meta.Col, 1)
		err := f.SetCellValue(name, col, meta.Value)
		if err != nil {
			return err
		}
	}
	refValue := reflect.ValueOf(obj)
	for i := 0; i < refValue.Len(); i++ {
		objElem := reflect.ValueOf(refValue.Index(i).Interface()).Elem()
		for _, meta := range metaMap {
			// 从第二行开始插数据
			col := fmt.Sprintf("%s%d", meta.Col, i+2)
			err := f.SetCellValue(name, col, objElem.FieldByName(meta.Key).Interface())
			if err != nil {
				return err
			}
		}
	}
	return nil
}

```


- https://github.com/qax-os/excelize/issues/1540