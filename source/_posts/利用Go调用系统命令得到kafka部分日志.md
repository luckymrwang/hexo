title: 利用Go调用系统命令得到kafka部分日志
date: 2016-07-26 19:20:17
tags: [Go]
---

*自己加的一个利用Go调用系统命令得到kafka部分日志的函数*

```go
// @router /get_user_items_log
func (c *QueryController) GetUserItemsLog() {
        t := time.Now().AddDate(0, 0, -1)
        dateStr := t.Format("2006-01-02 00:00:00")
        fmt.Println(dateStr)
        date, _:= time.Parse("2006-01-02 00:00:00", dateStr)
        timestamp := date.Unix()
        timestr := strconv.FormatInt(timestamp, 10)

        consumer := &models.Consumer{}
        offset := consumer.GetOffset(timestr)
        cmdstr := fmt.Sprintf(`/data/plattech/kafka_2.9.2-0.8.1.1/bin/kafka-simple-consumer-shell.sh --broker-list 10.251.192.137:9092 --topic game_server --partit
ion 0 --offset %d --no-wait-at-logend | grep '"type":"item"' > kafka_raw.txt`, offset)
        cmd := exec.Command("/bin/sh", "-c", cmdstr)
        err := cmd.Run()
        if err != nil {
                panic(err)
        } else {
                c.Ctx.WriteString("Success.")
        }
}
```