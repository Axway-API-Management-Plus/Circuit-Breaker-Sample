# Circuit-Breaker-Sample

## Description
In development, a circuit breaker is a design pattern to encompass logic to circumvent reoccuring failures by changing the flow when a process, external system, etc. is failing. Rather than continuously attempting to utilize the failing resource, the circuit breaker will implement a logic change or fail over to a fault handler rather than continuing down the same path until the issue is resolved or for a set period of time.

This is a sample policy framework of how one might implement this within the Axway API Gateway using only OOTB filters with a minimal amount of extended logic built into the Scripting Filter.

## Terms

- ***Circuit Box:*** A failure cache that contains a rolling count for each circuit that is used to make processing/flow decisions.
- ***Circuit:*** A failure point tracked in the Circuit Box with a unique message key for each circuit that iterates upon each failure.

## Prerequisites

- ***Implement the Circuit Box:*** In Policy Studio, create a new cache under Environment Configuration --> Libraries. Depending on your environment and needs the type of cache you create will differ, but generally in highly available, distributed architecture this will use a distributed cache. Configure your time to live. Effectively this is how long from the last error that occurred you want the circuit to remain open.

A local cache for this purpose is defined in the policy artifacts in the /src folder.

##  Circuit Breaker Policy

This policy was built with the intent of being a framework for a Global Request Policy for all API Gateway traffic. As messages come in, this policy will check the status of the Circuit Breakers within the Circuit Box, and handle the policy flows dynamically built on your configuration for how to handle partially open, fully open, or closed Circuits.

- ***Starting Filter Placeholder - Not Required:*** This is an Extract REST Attributes filter. It is not required for this policy, but I often use it as the starting filter for policies, as it should always pass and has value for making additional attributes available if required.

- ***Set Variables for Circuit Breaker Names and Break Values:*** This Copy/Modify Attribute filter is being used to declare attributes that will be used to identify the different Circuit Breakers we will be tracking and the soft/hard limits that we wish to use to implement additional logic.

In this example policy, httpCircuitBreaker is being used as the message key within the Circuit Box cache to track HTTP routing failures for our sample service. The Max and Soft limits are compared against the cached value for the Circuit Breaker to determine if alternate logic is required, or if the Circuit should be open to prevent additional traffic. In this policy the soft limit is only declared to provide an example placeholder for additional logic. The hard limit will be utilized to route to a fault handler/alternative response when hit, rather than routing to the standard processing policy.

Additional attributes would need to be populated here for each Circuit Breaker you have implemented.


- ***IF EXISTS - Get HTTP Circuit Breaker Count:*** This filter is a cache lookup to see if the Circuit Breaker in question has a stored count. If no failures have occurred relative to your time to live value, then this will not return a result and your normal policy logic will resume in place of the "IF Circuit Breaker Logic Passes - THEN Continue Processing".

With this implementation, you would need to repeat this filter for each Circuit Breaker you wished to check before you continued processing. Additional logic could be implemented to associated different Circuit Breaker checks with different policies, URIs, users, etc. if desired as well.

- ***Compare Circuit Breaker Soft/Max Values to Current Count:*** This Scripting Filter will compare the soft/hard attributes you declared earlier against the current counts that were found in the Cache Lookup filter. If the count for the Circuit Breaker is greater than or equal to the the hard/soft limit, additional logic can be executed. This could be done with the Compare Attribute filter, but you would need one for each Circuit and limit to determine the logic. With this implementation, you could have repeatable outcomes based on limits being hit across any number of Circuit Breakers.

In this example, we check to see if the HTTP Circuit Breaker Count has met or exceeded the hard limit set in the Compare Attributes filter (5). If it does, the script returns false to follow the fault handler/alternative logic path. If the count existed but has not hit the limit, it will process the request as normal.

This logic could be extended by declaring a new variable with a value to allow for contextual policy of additional routes or extra logic. For example, maybe a limitBreak variable is set to none, soft, or hard. This could then be returned to policy and used with the Switch on Attribute Value filter to call different policies to implement more advanced Circuit Breaker logic such as a half open breaker.

- ***Implement Circuit Breaker Logic and Reflect:*** This is mostly just a place holder for the logic you wish to implement if a Circuit is Broken. Likely in your environment this would be a separate policy that is invoked with the Policy Shortcut filter that addresses your logic for the Circuit Break scenario.

##  Sample HTTP Client Policy

- ***Set/Initialize Variables for HTTP Circuit Breaker:*** This Copy/Modify Attribute filter allows you to declare multiple attributes at once. This creates a 0 count attribute for the Circuit Breaker for the policy, as well as a variable for the name of the Circuit Breaker that failure counts will be tracked against.

- ***Connect to Non-Existent Endpoint to Cause Failure and Iterate Count:*** This filter is a Connect to URL filter that intentionally hits a non-existent endpoint on the API Gateway. This is used for demo purposed to cause a failed request that will iterate our Circuit Breaker count so that in our sample global policy the limit will eventually be hit. If the filter fails to connect, it will immediately drop to the counter iteration. If it passes it will call the Compare Attribute filter for further analysis.

- ***IF Result is not Desired/Expected - THEN Implement Circuit Breaker Count Iteration Logic:*** This is a Compare Attribute filter. The sample here assumes the only desirable outcome is a 200. In this sample policy, we will always get a 403 FORBIDDEN back as there is not an endpoint implemented on the API Gateway where the Connect to URL filter is looking. If a 200 were returned, this would end the policy flow as successful.

- ***Check if HTTP Circuit Breaker has Active Count:*** Assuming a failure to route or a failure for the Compare Attribute check, the Circuit Breaker count will need iterated. This Cache Lookup filter will check the named Circuit Breaker (used as the message key name) to see if a count exists.

- ***Iterate HTTP Circuit Breaker Count By One:*** If a count did not exist, the 0 count attribute that was initialized at the start will be iterated. If a count was found, it will replace that 0 placeholder and be iterated. Either way the count will increase by one and the updated attribute will be returned to policy in this Scripting Filter.

- ***Update HTTP Circuit Breaker Count:*** This Cache Attribute filter will use the Circuit Breaker name as the message key and update it with the most recent count, which was iterated in the last filter. This will also update the time to live within the Circuit Box and refresh the timer before it resets.

- ***Mock Response and Reflect:*** This is where the routing failure message handler would be. In this case we just used a simple mockup set message and reflect, which shows the HTTP Code captured and the current Circuit Breaker Count. This will iterate until it reaches the limit set in the global policy. When this limit is hit, the global policy will open the Circuit and handle the fault there, rather than routing back to this policy.
