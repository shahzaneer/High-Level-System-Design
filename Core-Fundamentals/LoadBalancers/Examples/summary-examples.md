# Cloud Load Balancers Comparison

| Feature      | AWS ALB                             | AWS NLB       | GCP Global HTTP(S) LB | GCP Network LB | Azure App Gateway | Azure Load Balancer | Azure Front Door |
| ------------ | ----------------------------------- | ------------- | --------------------- | -------------- | ----------------- | ------------------- | ---------------- |
| Layer        | L7                                  | L4            | L7                    | L4             | L7                | L4                  | L7               |
| Scope        | Regional                            | Regional      | Global                | Regional       | Regional          | Regional/Global     | Global           |
| Protocols    | HTTP/HTTPS, HTTP/2, WebSocket, gRPC | TCP, UDP, TLS | HTTP/S, HTTP/2, gRPC  | TCP, UDP       | HTTP/S, WebSocket | TCP, UDP            | HTTP/S           |
| Static IP    | No                                  | Yes           | Yes (Anycast)         | Yes            | No                | Yes                 | Yes (Anycast)    |
| WAF          | Via AWS WAF                         | No            | Via Cloud Armor       | No             | Built-in          | No                  | Built-in         |
| CDN          | Via CloudFront                      | No            | Built-in (Cloud CDN)  | No             | No                | No                  | Built-in         |
| Path Routing | Yes                                 | No            | Yes                   | No             | Yes               | No                  | Yes              |
| Autoscaling  | Automatic                           | Automatic     | Automatic             | Automatic      | Yes (v2)          | Automatic           | Automatic        |
