# Navigating the Frontier of Generative AI: Insights into API Gateway Design
In the rapidly evolving landscape of artificial intelligence, Generative AI has emerged as a game-changing technology with far-reaching implications. For the past 18 months, I've been on an exhilarating journey, immersing myself in the world of Generative AI, driven initially by personal curiosity and now fueled by both professional demands and an insatiable desire to push the boundaries of what's possible.

This document distills some of the key insights and challenges I've encountered while implementing Generative AI across various use cases, with a particular focus on API Gateway design. As people rush to harness the power of this transformative technology, understanding the intricacies of API architecture becomes crucial for building robust, scalable, and efficient Generative AI systems.

In the following sections, we'll dive deep into two critical aspects of Generative AI API Gateway design:

1. **API Flow Control & Size Constraints:** We'll explore the delicate balance of setting appropriate rate limits, optimizing queue polling rates, and determining input prompt size limits. These considerations are vital for managing expectations and implementing effective throttling mechanisms.

2. **API Response Models:** We'll examine the pros and cons of different response models, including REST, Async API, Server-Side Events, and WebSocket. Understanding the strengths and weaknesses of each approach is crucial for selecting the most suitable model for your specific use case.

By sharing my experiences and the conclusions I've drawn, I aim to provide you with valuable insights that will help you navigate the complex terrain of Generative AI API Gateway design.

## API Flow control & Size Constraints

Imagine you're at a busy restaurant. The kitchen can only handle so many orders at once, and there's a limit to how big each dish can be. This is similar to how APIs work – they have limits on how many requests they can handle and how much information each request can contain.

Over time, as APIs have become more common, we've gotten better at managing these limits. But there's a new challenge with Generative AI APIs, like those used for chatbots or text generation.

Traditional APIs count the number of requests, like counting how many dishes a kitchen makes. But Generative AI APIs are different – they care more about the total amount of words (or 'tokens') they process, like measuring the total weight of all ingredients used in a kitchen.

This difference means our usual ways of managing API limits don't work well for Generative AI. We need a new approach, but it's not as simple as it sounds. There are important questions about security, privacy, scalability, and complexity that we need to consider carefully. Let's explore some possible solutions

### 1. **Token Limiting at the API Gateway Level:**

Imagine a specialized maître d' at a restaurant entrance, adept at not only counting incoming guests but also estimating the total order size each party will request.

This method is straightforward if your API gateway supports this functionality. Here’s a step-by-step breakdown:

- The API gateway intercepts each incoming request.
- It goes beyond mere counting; the gateway decrypts the request and scrutinizes the content intended for the Large Language Model, particularly analyzing the prompt size.

This approach, however, introduces significant privacy and security considerations. Comparatively, it’s akin to the maître d' inspecting the contents of each guest’s takeout box before allowing them entry, which:

- May infringe on data privacy norms during transmission.
- Prematurely exposes the request’s content to the API Gateway provider, potentially before your own systems process it.
- Necessitates the formulation of enhanced Service Level Agreements (SLAs) to address these added security and privacy concerns, thereby adding complexity to your security framework.

Additionally, while this method effectively manages input tokens, it overlooks the combined rate limiting of both input and output tokens by the underlying model.

This doesn't imply that providers are inherently untrustworthy, but it does layer in additional complexity. You would need to engage in detailed discussions with your provider to ensure that data handling protocols during these operations meet your security and privacy standards.

### 2. **Crafting Your Own Token Gatekeeper:**

This approach involves developing a bespoke service that acts as a mediator between your API Gateway and the Large Language Model (LLM). Here’s how it unfolds:

- Requests seamlessly pass through the API Gateway, retaining their original security and privacy attributes.
- These requests are then directed to a purpose-built service that assesses and manages the token quota for each time window before they advance to the LLM.

Consider employing a straightforward rate limiting mechanism, like the leaky bucket algorithm, in this setup:

a) **The Always-On Approach:**

- Establish a dedicated virtual machine equipped with a memory-resident token bucket. This bucket initializes with a number reflecting the LLM's maximum token capacity.
- Each incoming request checks this token bucket. If the bucket has enough tokens, they are deducted, and the request is processed; if not, the request is denied.

This method is straightforward but struggles with scalability. Expanding this setup would necessitate deploying Kubernetes clusters, each with its own token bucket, potentially causing inconsistencies when there's only one LLM handling all requests.

b) **The Distributed Approach:**

- Shift to a global state by housing the token bucket in a centralized database or a caching system like Redis.
- This adjustment aids scalability and facilitates managing a higher volume of requests, but it also brings about challenges in ensuring transactional consistency, managing latency, and implementing distributed locks.

Both strategies introduce significant complexity to your system architecture, requiring a careful evaluation of the trade-offs between scalability, consistency, and performance.

**Additional Consideration:**

The primary focus of this token management strategy is on regulating inbound tokens, without addressing how the LLM considers both input and output tokens in its throttling mechanism. Integrating a system that also manages outbound tokens adds another layer of complexity and is a crucial factor in maintaining system balance and efficiency.

Each method must be precisely tailored to meet the specific operational demands and constraints of your application, highlighting the importance of a meticulous approach to determine the most fitting architecture.


### 3. Optimistic/ Greedy Token Allocation: A Mathematical Approach

This approach offers an elegant solution to token management for Large Language Models (LLMs), leveraging existing API gateway rate limiting mechanisms and asserting input and output token sizes in the function that calls the LLM. By modeling the problem mathematically, we can achieve a predictable design without adding unnecessary technical complexity.

#### The Model

Consider an API gateway integrating with a function (e.g., AWS Lambda) that calls an LLM with token limits. The system components work as follows:

- API gateway: Applies request rate limiting
- Lambda function: Enforces max input prompt token limit
- LLM call: Sets max_token field to impose output token limit

Given the LLM's total tokens-per-minute rate limit, we aim to calculate the optimal:

- Request rate limit (or queue polling rate)
- Input token size
- Output token size

#### Key Observations
1. LLMs typically have a consistent maximum output token limit (currently around 4096 tokens).
2. We can optimistically reserve a specific output token count based on requirements.


Now for the system to operate optimally:

$$
\text{Number of requests per minute} \times (\text{Number of input tokens per request} + \text{Number of output tokens per request}) = \text{Total token limit of the model per minute}
$$

For mathematical ease, let:

$$
\begin{aligned}
R &= \text{Number of requests per minute (enforced by API Gateway and it rate limiting methods)} \\
IN &= \text{Number of input tokens per request (enforced by the function after the API gateway)} \\
OUT &= \text{Number of output tokens per request} = 4096 \text{(given current models)} \\
T &= \text{Total token limit of the model per minute}
\end{aligned}
$$

Then the above formula becomes:

$$
R \times (IN + OUT) = T \tag{1}
$$

With constraints:

$$
\begin{aligned}
R &\leq \text{Maximum LLM provider API rate limit} \\
IN &\leq \text{Maximum LLM input token limit}
\end{aligned}
$$

From this, we can derive:


1. The optimal rate limit (R) for a given input size:

$$
R = \frac{T}{IN + OUT} \tag{2}
$$

2. The maximum input size (IN) for a given rate limit:

$$
IN = \frac{T}{R} - OUT \tag{3}
$$

Additional metrics:

1. Calculating Reserved Output Tokens

$$ \text{Reserved Output Tokens} = R \times OUT \tag{4} $$ 

2. Percentage of Total LLM Limit Reserved for Output

$$ \text{Percentage Reserved for Output} = \frac{R \times OUT}{T} \times 100% \tag{5} $$

This approach maps to existing API gateway rate limit control by providing a clear, mathematically-derived value for R via (2), which can be directly implemented as the gateway's rate limit. The function input size control is represented by IN via (3), which can be enforced within the function that processes requests after they pass through the API gateway. The output size constraint can be imposed on the model by sending the `max_tokens` field to the set by the limit to the model

From an implementation perspective, this method offers several advantages:

- **Simplicity:** It eliminates the need for complex token counting systems or distributed state management.
- **Clear constraints:** The system operates within well-defined boundaries (R and IN), making it easier to manage and predict behavior.
- **Efficiency:** By making an optimistic assumption about output tokens, it allows for maximum utilization of the LLM's capabilities without overcomplicating the control mechanism.
- **Flexibility:** The formulas can be easily adjusted if LLM token limits change in the future.

#### Tradeoffs

*Pros:*

- Simple and predictable
- Leverages existing API gateway features
- Easy to reason about and debug
- No additional infrastructure required

*Cons:*

- May over-reserve tokens in some scenarios
- Potential underutilization of LLM capacity

The Optimistic Token Allocation approach offers a balanced solution for many LLM applications, providing simplicity and effectiveness. However, specific use cases with high-security requirements or unique needs may benefit from alternative approaches. The choice ultimately depends on your application's scale, privacy requirements, and available resources.

> In the python notebook, `api-limits-design.ipynb`, I have implented these function and tabulated different constraints for given popular models if they are server by either API Gateway of a Queue polling mechanism (the idea remains the same)

#### Example scenarios

Let's explore this in some senarios

#### Scenario 1: High-frequency, Low-input Token Use Case for a Support Center Chatbot

**Context:**
We're implementing a chatbot system for a support center using the Retrieval-Augmented Generation (RAG) pattern. The system needs to handle a high volume of domain-specific questions efficiently.

**System Overview:**
1. An API gateway receives incoming queries.
2. Queries are routed to a handler function.
3. The handler function retrieves relevant context from a vector store.
4. The context and query are sent to a Large Language Model (LLM) for processing.
5. The LLM generates an appropriate response.

**Design Goals:**
- Handle up to 200 requests per minute.
- Optimize token usage within the LLM's constraints.
- Balance between input context, message history, and output generation.

**Given Parameters:**
- Model total token limit per minute (T): 1,000,000 tokens
- Maximum Model API request rate limt: 500 requests per minute
- Desired maximum rate limit (R): 200 requests per minute
- Fixed maximum output token (OUT): 400 tokens (approximately 300 words) - Changes to this will change all subsequent calculations

**Calculated Input Token Limit:**
Using the formula: IN = (T / R) - OUT
IN = (1,000,000 / 200) - 400 = 4,600 tokens

**Token Allocation Strategy:**
Total available input tokens per request: 4,600 (approximately 3,450 words)
Proposed allocation:
1. Current input query: 400 tokens (300 words)
2. Message history: 1,200 tokens (900 words)
3. Retrieved context: 3,000 tokens (2,250 words)

**Implementation Guidelines:**
1. API Gateway Configuration:
   - Set rate limit to 200 requests per minute.

2. Handler Function Design:
   - Limit input query to 400 tokens.
   - Maintain a message history of up to 1,200 tokens.
   - Retrieve and filter the most relevant 3,000 tokens of context from the vector store.

3. LLM Integration:
   - Combine input query, message history, and retrieved context.
   - Set the `max_tokens` parameter to 400 for the LLM response.

**Benefits of This Approach:**
- Maximizes the use of available tokens within the LLM's constraints.
- Provides a balance between current input, historical context, and retrieved information.
- Ensures consistent response times by limiting output token generation.

**Considerations:**
- The token allocation between input query, history, and context can be adjusted based on specific use case requirements.
- Regular monitoring and adjustment may be necessary to optimize performance and token usage.

This enhanced scenario provides a clearer explanation of the system's design, goals, and implementation strategy, making it easier for team members to understand and implement the chatbot system effectively.


## Conclusion

In conclusion, each approach has its merits and drawbacks. The choice depends on your specific use case, scale, privacy requirements, and available resources. The Optimistic/Greedy Token Allocation seems to offer a good balance of simplicity and effectiveness for many scenarios, but for applications with very specific needs or high-security requirements, one of the other approaches might be more suitable.
