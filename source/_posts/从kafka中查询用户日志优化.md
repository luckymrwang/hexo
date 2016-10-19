title: 从kafka中查询用户日志优化
date: 2016-10-18 17:04:37
tags: [Kafka, Go]
---

优化从`kafka`中取用户日志

最初是根据开始时间，结束时间从`kafka`提供的api中取出`json`数据，然后挨个解析判断是否符合条件。

由于数据量很大，这种方式会特别慢。

### 初步优化

通过`kakfa`官方提供的命令行工具，取出用户日志，然后再利用系统`grep`命令筛选出符合条件的数据，此时数据为`json`流，之后将`json`流数据循环解析过滤出最终数据

<!-- more -->
####（部分代码）展示：

```go
func (c *Consumer) GetUserLogs(bigAppId, zid, uid string, startTime, endTime int64) (msgs []map[string]interface{}) {
	startOffset, endOffset := c.GetStartEndOffsets(startTime, endTime)
	msgcnt := endOffset - startOffset

	cmdstr := fmt.Sprintf(`/kafka/bin/kafka-simple-consumer-shell.sh --broker-list %s --topic game_server --partition 0 --offset %d --max-messages %d | grep '"user_id":%s'`, brokerList, startOffset, msgcnt, uid)
	fmt.Println(cmdstr)
	out, err := exec.Command("/bin/sh", "-c", cmdstr).Output()
	helpers.CheckError(err)

	dec := json.NewDecoder(strings.NewReader(string(out)))

	for {
		var msg map[string]interface{}
		dec.UseNumber()
		if err := dec.Decode(&msg); err == io.EOF {
			break
		} else if err != nil {
			log.Fatal(err)
		}

		tm, err := msg["time"].(json.Number).Int64()
		if err != nil {
			log.Fatal(err)
		}

		if tm >= startTime && tm <= endTime {
			msgs = append(msgs, msg)
		}
	}

	return msgs
}
```

### 进一步优化

该优化是在上一步的基础上利用Go的并发特性，每10~100万条数据开一个进程，最后将数据汇总起来

（部分代码）展示：

```go
func (c *Consumer) GetUserLogs(bigAppId, zid, uid string, startTime, endTime int64) []map[string]interface{} {
	var wg sync.WaitGroup

	startOffset, endOffset := c.GetStartEndOffsets(startTime, endTime)

	ch := make(chan []map[string]interface{})

	stepLen := int64(100 * 10000)
	for i := startOffset; i <= endOffset; i = i + stepLen {
		fmt.Println(i, i+stepLen, endOffset)

		msgcnt := stepLen
		curStepLast := i + stepLen
		// 最后一次时需要修正下msgcnt
		if curStepLast > endOffset {
			msgcnt = (endOffset - startOffset) % stepLen
		}

		wg.Add(1)
		go func(from, msgcnt int64, uid string, ch chan []map[string]interface{}) {
			defer wg.Done()
			msgs := make([]map[string]interface{}, 0)
			cmdstr := fmt.Sprintf(`/kafka/bin/kafka-simple-consumer-shell.sh --broker-list %s --topic game_server --partition 0 --offset %d --max-messages %d | grep '"user_id":%s'`, brokerList, from, msgcnt, uid)
			fmt.Println(cmdstr)
			out, err := exec.Command("/bin/sh", "-c", cmdstr).Output()
			if err != nil {
				return
			}

			dec := json.NewDecoder(strings.NewReader(string(out)))

			for {
				var msg map[string]interface{}
				dec.UseNumber()
				if err := dec.Decode(&msg); err == io.EOF {
					break
				} else if err != nil {
					log.Fatal(err)
				}

				tm, err := msg["time"].(json.Number).Int64()
				if err != nil {
					log.Fatal(err)
				}

				if tm >= startTime && tm <= endTime {
					msgs = append(msgs, msg)
				}
			}

			ch <- msgs

		}(i, msgcnt, uid, ch)
	}

	go func() {
		wg.Wait()
		close(ch)
	}()

	mapCh := make([]map[string]interface{}, 0)
	for data := range ch {
		mapCh = append(mapCh, data...)
	}

	return mapCh
}
```
