### Authorization

Authorization is a complex area.

Most applications use a role-based authorization system today. To support RBAC (role-based access control)
it is necessary to authorize actions based on (multiple) roles a client may have.

#### Application Features

To simplify development of applications that use RBAC we assign a `feature` to each remote procedure
and topic. A feature is designated by an URI as unique identifier.

Features are logical groups of API endpoints which can be used by clients with the same permissions.

`
TODO: Example
`

#### Interaction between features and authroles

Features and authroles are orthogonal concepts. A feature can be used by many authroles and an authrole can use many features. Therefore features and authroles have an n-to-m mapping between them.

For performance reasons, it's recommended to implement the mapping as two-dimensional array/map or a similar concept.

#### Feature Definition

By default, RPCs are not assigned to any feature.

When RBAC is enabled, the first `Callee` to register a procedure for a particular URI MAY set a feature the procedure belongs to. If the feature is not set, the procedure is globally available to call.

This is done through setting

{align="left"}
        REGISTER.Options.feature|uri := <feature_uri>

For publications there is no explicit registration. Therefore it is necessary to claim a topic to ensure no unauthorized access will occur. If a topic is not claimed by any feature it is publicly available for publications and subscriptions.

A client can call the following procedure to claim a topic:

* `wamp.topic.claim(topic|uri, sub_feat|uri, pub_feat|uri)`

A client can claim a topic to have limitations only on the publication or subscription of the topic by passing `null` for `sub_feat` or `pub_feat`. If a topic claim exist for a given topic and a second should be added, the router has to check whether `pub_feat` and `sub_feat` belong to the same feature originally requested. If they belong to the same feature, it shall save the session ID the topic was claimed for to keep track on the number of claims for the topic. If a session leaves the router, remove the session ID from all topic claims belonging to this session ID. If a claim has no active sessions left, remove the claim entirely. A claim is only possible if NO client is subscribed to the topic to claim or another claim with the same parameters already exists.

#### Authorizing actions within the router using RBAC

The authorization workflow depends on the type of the message.

##### Authorizing CALLs

A CALL can be authorized as follows:
- Router gets the feature the registration belongs to
    - if no feature could be found, PERMIT the CALL
- Router gets the authentication roles allowed to use the feature
- Router checks whether the authrole of the `caller` matches any of the authroles fetched before
- If it matches, PERMIT the CALL, otherwise REJECT it

##### Authorizing PUBLICATIONs

A PUBLICATION can be authorized as follows:
- Router gets the claim for the topic URI
    - if no claim for the topic exists PERMIT the PUBLICATION
- Router gets the authentication roles allowed to use feature in `claim.pub_feat`
- Router checks wether the authrole of the `publisher` matches any of the authroles fetched before
- If it matches, PERMIT the PUBLICATION, otherwise REJECT it

##### Authorizing SUBSCRIPTIONs

A SUBSCRIPTION can be authorized as follows:
- Router gets all claims, iterates through them, checks whether the topic of the claim matches the topic of the subscription
    - if it does not match, move on to the next claim
- Router gets the authentication roles allowed to use feature in `claim.sub_feat`
- Router checks whether the authrole of the `subscriber` matches any of the authroles fetched before.
    - if it does not match, REJECT the SUBSCRIPTION
    - if it matches, move on to the next claim
- If all claims matched, PERMIT the SUBSCRIPTION

#### Desired workflow:


1. Developer configures the router to allow access to the feature `com.example.administration` to members of the `admin` authrole.
2. Client A wants to register `com.example.user.create` using the feature URI `com.example.administration`.
3. Client B wants to call `com.example.user.create`
    1. Router checks whether B has any of the authroles required to use the feature  `com.example.administration`.
    2. If B has features, permit the call
    3. Otherwise, reject the call with a `wamp.error.not-authorized` error
