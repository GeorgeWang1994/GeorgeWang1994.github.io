


# Go语言中的Ticker实现

## 背景



最近在工作中需要实现起一个异步不停的去查询容器的服务的状态是否好了的功能，所以需要写一个`Ticker`，而且这个`Ticker`还得提供超时的功能。



## 实现



### 维护操作数组



该方法的思路就是，维护一个可以操作`服务ID`的数组，当定时器执行的时候遍历数组，并且维护服务ID和失败次数的映射，当该失败次数到达5次了以后，则将其从操作数组中移除掉，不再执行。但是，并不是适用于当前我需要实现的场景，**因为我并不知道需要查询的每一个服务具体的执行时间是多少以及有多少个服务，也就是说我遍历查询全部的服务（不确定有多少，可能多也可能少）的时间是无法确定的**。因此放弃了这种方法。



```go
package main

import (
	"fmt"
	"time"
)

// 查询服务是否好了
func QueryService(serviceID string) error {
	if serviceID == "a" {
		fmt.Printf("fail: %v\n", serviceID)
		return fmt.Errorf("error")
	}

	fmt.Printf("success: %s\n", serviceID)
	return nil
}


// 在一个ticker中遍历所有的容器的状态
func DoTickerWork(serviceList []string, serviceMap map[string]map[string]string, timeout <-chan time.Time) {
	t := time.NewTicker(5 * time.Second)
	done := make(chan bool, 1)
	failMap := make(map[string]int, 0)
	go func() {
		for {
			select {
			case <-t.C:
				if len(serviceList) == 0 {
					close(done)
					return
				}

				for i := 0; i < len(serviceList); i++ {
					if failNum, exist := failMap[serviceList[i]]; exist && failNum >= 5 {
						serviceList = append(serviceList[:i], serviceList[i+1:]...)
						i--
						continue
					}

					serviceInfo, exist := serviceMap[serviceList[i]]
					if !exist {
						continue
					}

					err := QueryService(serviceInfo["id"])
					if err == nil {
						serviceList = append(serviceList[:i], serviceList[i+1:]...)
						i--
						fmt.Printf("serviceList :%v\n", serviceList)
						continue
					} else {
						if failNum, exist := failMap[serviceList[i]]; exist {
							failMap[serviceList[i]] = failNum + 1
						} else {
							failMap[serviceList[i]] = 1
						}
					}
				}
			case <-timeout:
				close(done)
				return
			}
		}
	}()
	<-done
	return
}


func main() {
	timeout := time.After(100 * time.Second)
	serviceList := make([]string, 0)
	serviceMap := make(map[string]map[string]string, 0)
	serviceList = append(serviceList, "a", "b", "c", "d")

	for _, serviceKey := range serviceList {
		serviceMap[serviceKey] = map[string]string {
			"id": serviceKey,
		}
	}

	fmt.Printf("map is %v\n", serviceMap)
	DoTickerWork(serviceList, serviceMap, timeout)
}

```



具体的结果如下：

![image](https://wx1.sinaimg.cn/large/007FyU7Tly1g3sotwc87hj30ok0c4mxy.jpg)



### 每个查询都起一个异步



不像上一个方法，将所有的查询放到一个异步里做，而是每一个查询都起一个异步去查询，每个查询的时间也是可以把控的，而且在查询以后的操作也容易进行扩展，因此最后采用这个方案。



```go
package main

import (
	"fmt"
	"time"
)



func DoTickerWork(servicepotID string, timeout <-chan time.Time) {
	t := time.NewTicker(1 * time.Second)
	done := make(chan bool, 1)
	go func() {
		for {
			select {
			case <-t.C:
				err := QueryService(servicepotID)
				if err == nil {
					close(done)
					return
				}
			case <-timeout:
				close(done)
				return
			}
		}
	}()
	<-done
	return
}

func testGo(serviceVal string)  {
	timeout := time.After(5 * time.Second)
	go DoTickerWork(serviceVal, timeout)
}


func main() {
	serviceList := make([]string, 0)
	serviceList = append(serviceList, "a", "b", "c", "d")
	for _, serviceVal := range serviceList {
		go testGo(serviceVal)
	}

	<- time.After(10000 * time.Second)
}
```



具体的结果如下：



![image](https://wx4.sinaimg.cn/large/007FyU7Tly1g3sp8xtfikj30ey07idg2.jpg)

