# Concept
	- Perfect negotiation makes it possible to seamlessly and completely separate the negotiation process from the rest of your application's logic
	- The same code is used for both the caller and the callee, so there's no repetition or otherwise added levels of negotiation code to write.
	- Perfect negotiation works by assigning each of the two peers a role to play in the negotiation process that's entirely separate from the WebRTC connection state:
		- A **polite** peer, which uses ICE rollback to prevent collisions with incoming offers. A polite peer, essentially, is one which may send out offers, but then responds if an offer arrives from the other peer with "Okay, never mind, drop my offer and I'll consider yours instead."
		- An **impolite** peer, which always ignores incoming offers that collide with its own offers. It never apologizes or gives up anything to the polite peer. Any time a collision occurs, the impolite peer wins.
- # Implement
	- ## **Handling the negotiationneeded event**
	  ```javascript
	  let makingOffer = false;
	  
	  pc.onnegotiationneeded = async () => {
	    try {
	      makingOffer = true;
	      await pc.setLocalDescription();
	      signaler.send({ description: pc.localDescription });
	    } catch (err) {
	      console.error(err);
	    } finally {
	      makingOffer = false;
	    }
	  };
	  ```
	  The set description is either an answer to the most recent offer from the remote peer or a freshly-created offer if there's no negotiation underway. Here, it will always be an `offer`, because t**he negotiationneeded event is only fired in `stable` state**.
	  
	  We set a Boolean variable, `makingOffer` to `true` to mark that we're preparing an offer. To avoid races, we'll use this value later instead of the signaling state to determine whether or not an offer is being processed because the value of `signalingState` changes asynchronously, introducing a glare opportunity.
	- ## **Handling incoming ICE candidates**
	  ```javascript
	  pc.onicecandidate = ({ candidate }) => signaler.send({ candidate });
	  ```
	  The `RTCPeerConnection` event `icecandidate`, which is how the local ICE layer passes candidates to us for delivery to the remote peer over the signaling channel.
	  
	  This takes the `candidate` member of this ICE event and passes it through to the signaling channel's `send()` method to be sent over the signaling server to the remote peer.
	- #### **Handling incoming messages on the signaling channel**
	  
	  ```javascript
	  let ignoreOffer = false;
	  
	  signaler.onmessage = async ({ data: { description, candidate } }) => {
	    try {
	      // It's either an offer or an answer sent by the other peer.
	      if (description) {
	        const offerCollision =
	          description.type === "offer" &&
	          (makingOffer || pc.signalingState !== "stable");
	  
	        // If we're the impolite peer, and we're receiving a colliding offer, we return without setting the description
	        ignoreOffer = !polite && offerCollision;
	        if (ignoreOffer) {
	          return;
	        }
	  
	        // If we're the polite peer, and we're receiving a colliding offer, we don't need to do anything special, because our existing offer will automatically be rolled back in the next step
	        await pc.setRemoteDescription(description);
	        if (description.type === "offer") {
	          // setLocalDescription() to automatically generate an appropriate answer in response to the received offer
	          await pc.setLocalDescription();
	          signaler.send({ description: pc.localDescription });
	        }
	      // It's an ICE candidate received from the remote peer as part of trickle ICE
	      } else if (candidate) {
	        try {
	          // The candidate is destined to be delivered to the local ICE layer by passing it into addIceCandidate()
	          await pc.addIceCandidate(candidate);
	        } catch (err) {
	          //  If an error occurs and we've ignored the most recent offer, we also ignore any error that may occur when trying to add the candidate
	          if (!ignoreOffer) {
	            throw err;
	          }
	        }
	      }
	    } catch (err) {
	      console.error(err);
	    }
	  };
	  ```
	  Each time a message arrives from the signaling server invokes `onmessage` event.
	-