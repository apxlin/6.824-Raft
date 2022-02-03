# 6.824-Raft
Raft note

There are three roles in Raft algorithm, Follower, Candidate, Leader, each node store currentTerm, votedFor and log.
There are Two RPC in this protocol , RequestVote and AppendEntries.

When raft clusters start, each node perform a series of initialization (currentTerm is set to 0, votedFor is set to -1, and the initial state is Follower)

If the election time exceeds and does not receive the AppendEntries message from the current leader or the RequestVote message that agrees to the election of the candidate, the Follower will be converted to the Candidate state.

When a node step into candidate state, its currentTerm will increased 1, votedFor is set to the current node, random timer will reset and RequestVote messages will send to all other nodes.

Candidates will remain in the candidate status until one of the following three situations occurs
First, With the agreement of majority of the nodes, the node is elected as the leader and changes its state to the leader
Second, Received the AppendEntries message from another node indicates that it is the leader. If the current candidate node thinks that the leader is legal (term> currentTerm in the message carrying parameters), it will switch itself to the follower state.
Third, If the majority of votes are not received before the timer expires, a new round of elections will start.

The detail of RequestVote.
When a node receives a RequestVote message, it will make the following judgments:

If the term carried by the RPC smaller than the node’s currentTerm, it will return currentTerm and reject the voting request: (currentTerm, false), this node will keep the current node state unchanged.

If the term carried by the RPC equal to currentTerm, here are two siutaions,
if the votedFor of the voting node is not empty and is not equal to the candidateId carried by the parameter, the current term is returned and the voting request is rejected, the current node status remains unchanged; 
if voteFor is empty Or it is equal to candidateId, the node will agree to the voting request: (currentTerm, true), and convert the current node to Follower, reset the timer, and set voteFor to candidateId.

If the term carried by the RPC bigger than currentTerm, first set the voting node currentTerm to carried term, set voteFor to empty and switch to Follower state, reset the timer, and enter the next step of judgment:
If the term of the last log of the voting node bigger than lastLogTerm carried by the RPC, it will return the updated term and reject the voting request:
If the term of the last log of the voting node equal to lastLogTerm carried by the RPC, we will compare the Index of the last log of the node with the lastLogIndex carried by the RPC;
if the Index of the last log of the node bigger than lastLogIndex, return to the updated term and refuse to vote Request, otherwise agree to the voting request.

If the term of the last log of the voting node smaller than lastLogTerm carried by the RPC, set voteFor equal to candidateId and agree to the voting request

So from such RPC, we will elect a leader form candidate.

When a node step into leader state, it will Initialize the nextIndex and matchIndex for all other nodes.
Regularly send AppendEntries messages (heartbeat messages) to each node to maintain its leadership.

The detail of AppendEntries.
When a node receives a AppendEntries message, it will make the following judgments:

If the term carried by the RPC smaller than the node’s currentTerm, it will return currentTerm and reject the request: (currentTerm, false), this node will keep the current node state unchanged.

Otherwise, we need a further judgement:
If the current node log[] structure does not contain a log at the prevLogIndex which carried by the RPC, then reject this request
If the current node log[] structure contains a log at the index of prevLogIndex but the term of the log is not equal to prevLogTerm, reject this request
Otherwise, store this log.
When store this log, If the new log entry conflicts with the existing log entry, you should delete the existing log entry and the log entries after it, and then add the entries carried by the RPC to the current node log.

When leader receive the results of AppendEntries request, it will make the following judgments:

If the return term bigger than its currentTerm, set currentTerm equal to return term, reset the random timer, and convert itself to Follower state
If the returned message is false (due to log inconsistency), subtract 1 from the value of the node in nextIndex[] and resend the message
If the returned message is true, update the nextIndex and matchIndex values of the node
