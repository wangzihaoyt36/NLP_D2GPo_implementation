# Introduction to the evaluation interface
The simultaneous translation models from the shared task participants are evaluated under a server-client protocol. The participants are required to plug in their own model API in the protocol, and submit a Docker image.

## Server-Client Protocol
An server-client protocol that will be used in evaluation. For example, when a *wait-k* model (k=3) translate the English sentence "Alice and Bob are good friends" to Genman sentence "Alice und Bob sind gute Freunde." , the evaluation process is shown as following figure. 

Every time the client needs to read a new state (word or speech chunk), a "GET" request needs to be sent to the server. Whenever a new token is generated, a "SEND" request with the predicted word (untokenized word) will be sent to the server immediately. The server can hence calculate both the latency and BLEU score of the sentence.

### Server
The server code is provided and can be set up locally for development purposes. For example, to evaluate a text simultaneous test set,

```shell

  python fairseq/examples/simultaneous_translation/eval/server.py \
    --hostname local_host  \
    --port 1234 \
    --src-file SRC_FILE \  
    --ref-file REF_FILE  \  
    --data-type text \
```
The state that server sent to client is has the following format
```json
{
  'sent_id': Int,
  'segment_id': Int,
  'segment': String
}
```

### Client
The client will handle the evaluation process mentioned above. It should be out-of-box as well. The client's protocol is as following table

|Action|Content|
|:---:|:---:|
|Request new word / utterence| ```{key: "Get", value: None}```|
|Predict word "W"| ```{key: "SEND", value: "W"}```|



The core of the client module is the agent, which needs to be modified for different models accordingly. The abstract class of agent is as follow, the evaluation process happens in the `decode()` function. 
```python
class Agent(object):
    "an agent needs to follow this pattern"
    def __init__(self, *args, **kwargs):
        ...

    def init_states(self):
        # Initializing states
        ...

    def update_states(self, states, new_state):
        # Update states with given new state from server
        # TODO (describe the states)
        ...

    def finish_eval(self, states, new_state):
        # Check if evaluation is finished
        ...
    
    def policy(self, state: list) -> dict:
        # Provide a action given current states
        # The action can only be either
        # {key: "GET", value: NONE} 
        # or
        # {key: "SEND", value: W}
        ...

    def reset(self):
        # Reset agent
        ...
        
    def decode(self, session):
        
        states = self.init_states()
        self.reset()      

        # Evaluation protocol happens here
        while True:
            # Get action from the current states according to self.policy()
            action = self.policy(states)

            if action['key'] == GET:
                # Read a new state from server
                new_state = session.get_src()
                states = self.update_states(states, new_state)

                if self.finish_eval(states, new_state):
                    # End of document
                    break 
                
            elif action['key'] == SEND:
                # Send a new prediction to server
                session.send_hypo(action['value'])
                
                # Clean the history, wait for next sentence
                if action['value'] == DEFAULT_EOS:
                    states = self.init_states() 
                    self.reset()
            else:
                raise NotImplementedError

 
```
Here an implementation of agent of text [*wait-k* model](../eval/agent.py). Notice that the tokenization is not considered.

## Quality
The quality is measured by detokenized BLEU. So make sure that the predicted words sent to server are detokenized. An implementation is can be find [here](../eval/agent.py)

## Latency
The latency metrics are 
* Average Proportion
* Average Lagging
* Differentiable Average Lagging
Again, they will also be evaluated on detokenized text.