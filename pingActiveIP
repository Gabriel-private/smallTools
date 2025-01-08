package main

import (
	"fmt"
	"golang.org/x/net/icmp"
	"golang.org/x/net/ipv4"
	"net"
	"os"
	"sync"
	"time"
)

const packetSize = 64

func sendICMPRequest(targetIP string, wg *sync.WaitGroup, resultChan chan<- string, timeout time.Duration) {
	defer wg.Done()

	conn, err := net.Dial("ip4:icmp", targetIP)
	if err != nil {
		fmt.Println("Error creating socket:", err)
		return
	}
	defer conn.Close()
	
	message := []byte("Hello ICMP")
	echoRequest := icmp.Message{
		Type: ipv4.ICMPTypeEcho, // Echo请求
		Code: 0,
		Body: &icmp.Echo{
			ID:   os.Getpid() & 0xffff, // 用进程ID做ID
			Seq:  1,
			Data: message,
		},
	}

	data, err := echoRequest.Marshal(nil)
	if err != nil {
		fmt.Println("Error marshaling ICMP request:", err)
		return
	}

	_, err = conn.Write(data)
	if err != nil {
		fmt.Println("Error sending ICMP request:", err)
		return
	}

	err = conn.SetReadDeadline(time.Now().Add(timeout)) // 设置超时时间
	if err != nil {
		fmt.Println("Error setting read deadline:", err)
		return
	}

	reply := make([]byte, packetSize)
	n, err := conn.Read(reply)
	if err != nil {
		fmt.Println("No reply received:", err)
		return
	}

	icmpReply := reply[20:n]
	receivedMessage, err := icmp.ParseMessage(1, icmpReply) 
	if err != nil {
		fmt.Println("Error parsing ICMP reply:", err)
		return
	}

	if receivedMessage.Type == ipv4.ICMPTypeEchoReply {
		fmt.Println("Received ICMP Echo Reply from", targetIP)
		resultChan <- targetIP
	} else {
		fmt.Println("Received unexpected ICMP response. Type:", receivedMessage.Type)
	}
}

func incIP(ip net.IP) {
	for j := len(ip) - 1; j >= 0; j-- {
		ip[j]++
		if ip[j] > 0 {
			break
		}
	}
}

func main() {
	targetIPRange := "8.8.8.0/24"
	_, ipNet, err := net.ParseCIDR(targetIPRange)
	if err != nil {
		fmt.Printf("Could not parse CIDR: %v\n", err)
		return
	}

	var wg sync.WaitGroup
	resultChan := make(chan string, 100) 

	timeout := 2 * time.Second

	for ip := ipNet.IP.Mask(ipNet.Mask); ipNet.Contains(ip); incIP(ip) {
		wg.Add(1)
		go sendICMPRequest(ip.String(), &wg, resultChan, timeout)
	}

	go func() {
		wg.Wait()
		close(resultChan)
	}()

	aliveCount := 0
	for range resultChan {
		aliveCount++
	}
	fmt.Printf("Number of alive IPs in the %s : %d\n", targetIPRange, aliveCount)
}
