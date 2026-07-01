# Migration From The Hackathon Prototype

This repo contains the production relay and receiver path for SafeHaven v1. It
replaces the hackathon WebRTC/signaling setup with a single server-mediated
WebSocket flow.

## Relay Changes

- Removed SDP, ICE, offer/answer, and per-viewer peer routing.
- Replaced peer signaling with token-keyed rooms.
- Added in-memory event replay for late receivers.
- Added retained latest video keyframe for faster video resync.
- Added sender presence notifications when receivers join or leave.
- Kept relay storage RAM-only with no payload logging.

## Receiver Changes

- Receiver source now lives inside this repo under `receiver/`.
- The production receiver is built during relay deploy and served by the relay.
- The browser connects to same-origin `/ws`.
- Video is decoded with WebCodecs and painted to a canvas.
- Audio is raw PCM scheduled through Web Audio.
- GPS, tier, and AI-label state are reconstructed from protocol events.

## Removed Pieces

- WebRTC media tracks
- STUN/TURN assumptions
- Hypercore and Hyperswarm transport
- Browser Babel-in-HTML runtime
- Separate production hosting requirement for the receiver
