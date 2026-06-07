# WebRTC

WebRTC (Web Real-Time Communication) enables peer-to-peer audio, video, and data sharing directly between browsers without plugins or intermediate servers. It's the technology behind Google Meet, Zoom (web), Discord video, and telehealth platforms. The key innovation: browsers can establish direct peer connections, reducing latency and server bandwidth costs compared to relay-based approaches.

WebRTC is not purely peer-to-peer in practice—signaling servers and TURN relays are required for connection establishment and NAT traversal. The architecture involves three main APIs: MediaStream (camera/microphone access), RTCPeerConnection (the peer connection), and RTCDataChannel (arbitrary data).

## Architecture

```
                    SIGNALING SERVER
                   (WebSocket / REST)
                 ┌─────────┼─────────┐
                 │                   │
           [Client A]  ──Media──→  [Client B]
                     (PeerConnection)
                     
             If direct connection fails:
           [Client A]  ──→  [TURN Server]  ──→  [Client B]
                        (Relay - last resort)
```

1. **Signaling**: Exchange session descriptions (SDP) via WebSocket server—offer/answer model
2. **ICE (Interactive Connectivity Establishment)**: Find the best connection path
3. **NAT Traversal**: STUN servers discover public IP; TURN servers relay if direct fails
4. **Media/Data**: Direct peer-to-peer once connection established

## Key Concerns for Architects

- **Signaling server must scale**: Every connection starts here. Use stateless WebSocket servers with Redis pub/sub backplane
- **TURN server cost**: TURN relays media traffic. At scale, TURN bandwidth cost dominates. Cloudflare, Twilio, or self-hosted coturn
- **SFU vs P2P for multi-party**: Peer-to-peer mesh doesn't scale (N connections per participant). Use Selective Forwarding Unit (SFU)—each participant sends one stream; SFU forwards to others
- **Codec negotiation**: VP8/VP9 (free), H.264 (hardware acceleration, patent-encumbered), AV1 (next-gen, free)

## Summary

WebRTC enables real-time communication directly between browsers. The signaling server and TURN infrastructure are the architectural concerns—the peer-to-peer media itself is handled by the browser. For multi-party calls beyond 3-4 participants, an SFU (Selective Forwarding Unit) is required to avoid the N-squared connection problem.
