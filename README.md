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

Picture this as having a specialized maitre d' at the restaurant entrance who not only counts guests but also estimates the total amount of food each party might order.

This approach is the most straightforward, provided your platform offers an API gateway service with this feature. Here's how it would work:

- The API gateway intercepts each request, as it's designed to do.
- In addition to counting requests and overall request size, it decrypts the request and examines the field containing the prompt for the Large Language Model.

However, this method raises significant privacy concerns. It's like having the maitre d' open each guest's takeout container before they enter the restaurant. This approach:

- Potentially violates the principle of data privacy during transit.
- Exposes the request content to the API Gateway provider before it reaches your implemented code.
- Raises security and privacy questions, necessitating additional Service Level Agreements (SLAs) and complicating an already intricate security plan.

While this doesn't necessarily mean the provider can't be trusted, it introduces another layer of complexity. You'd need to engage in further discussions with your provider and seek assurances about how the data is handled during this operation.

### 2. **Crafting Your Own Token Gatekeeper:**

In this approach, you're the head chef who must carefully monitor and control the flow of ingredients (tokens) while ensuring the kitchen (API) runs smoothly. Here's how it might work:

- Requests pass through the API Gateway untouched, preserving their security and privacy.
- Before reaching the Large Language Model (LLM), requests are routed to a service you've designed and manage.
- This service determines the remaining token quota for the current time window.

Now, let's consider implementing a simple leaky bucket algorithm (a simple rate limiting algorithm) in this scenario:

a) The Always-On Approach:

- You set up a dedicated virtual machine with allocated memory in which you implement the token bucket. Think of it as a just a piece of code which intialises a number in memory representing the LLM maximum token limit.
- All requests check the "token bucket" in memory when coming in only.
- If the bucket is empty, the request is rejected.
- If not, the required tokens are subtracted, and the request proceeds to the LLM.

However, this system lacks scalability. Scaling would require Kubernetes clusters, and each cluster would have its own in-memory bucket, leading to inconsistencies as we only have one LLM serving all these requests.

b) The Distributed Approach:

- You use a global state, storing the "token bucket" in an external database or Redis.
- This allows for easier scaling but introduces new challenges:
  - Ensuring consistency across multiple requests (ACID compliance)
  - Managing potential latency issues
  - Implementing distributed locking mechanisms

It's akin to having a central ingredient inventory system for all your restaurant branches, but now you need to ensure real-time accuracy and prevent conflicts when multiple chefs access it simultaneously.

Both approaches add complexity to your system architecture and require careful consideration of trade-offs between scalability, consistency, and performance. Like a master chef balancing flavors in a complex dish, you'll need to fine-tune your approach based on your specific needs and constraints.

**One more thing this approach misses out on** is the fact that the LLM takes into account both the input and output token for throttling mechanism. This approach can take into account only the inbound token, managing the outbound tokens and adding that into the mix an even bigger challenge.

> **Personal Experience:** I personally have gone down this rabbit-hole where I implemented similar throttling mechanism and have experienced it in all it complexity. I implemented a queue -> (token rate limiting lambda + dynamodb + dynamodb locking mechanism) -> queue -> LLM. It works well now but I would not recommend ever implementing it and I will be moving away from it to the idea explained next

### 3. Optimistic/ Greedy Token Allocation: A Mathematical Approach

This approach offers an elegant solution to the token management challenge, drawing inspiration from observed patterns in Large Language Model (LLM) behavior.

The key observation is that regardless of input size, LLMs typically have a consistent maximum output token limit (currently around 4096 tokens). This consistency allows us to make robust assumptions and set parameters accordingly.

Here's how it works:

1. For each LLM input, we optimistically reserve the maximum output token count (e.g., 4096 tokens). This can be lowered based on the use case as well. We are making an assumption in our system.
2. We leverage the existing request-based rate limiting mechanisms of API gateways.

The system operates optimally when:

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

This formula allows us to calculate:

1. The optimal rate limit (R) for a given input size:

$$
R = \frac{T}{IN + OUT} \tag{2}
$$

2. The maximum input size (IN) for a given rate limit:

$$
IN = \frac{T}{R} - OUT \tag{3}
$$

Based on the above formulae, we can derive some more helpful metrics

**Formula 1:** Calculating Reserved Output Tokens

$$ \text{Reserved Output Tokens} = R \times OUT \tag{4}$$ 

**Where:**
R = Requests per minute (rate limit) OUT = Output tokens per request (fixed at 4096)

**Explanation:** This formula calculates the total number of tokens reserved for output based on the rate limit and the fixed output token limit. It assumes that each request will use the maximum allowed output tokens.

**Formula 2:** Percentage of Total LLM Limit Reserved for Output

$$ \text{Percentage Reserved for Output} = \frac{R \times OUT}{T} \times 100% $$

**Where:** R = Requests per minute (rate limit) OUT = Output tokens per request (fixed at 4096) T = Total model token limit per minute

**Explanation:** This formula calculates the percentage of the total LLM token limit that is being reserved for output tokens. It divides the total reserved output tokens by the total model token limit and converts it to a percentage.

This approach maps to existing API gateway rate limit control by providing a clear, mathematically-derived value for R via (2), which can be directly implemented as the gateway's rate limit. The function input size control is represented by IN via (3), which can be enforced within the function that processes requests after they pass through the API gateway.

From an implementation perspective, this method offers several advantages:

- **Simplicity:** It eliminates the need for complex token counting systems or distributed state management.
- **Clear constraints:** The system operates within well-defined boundaries (R and IN), making it easier to manage and predict behavior.
- **Efficiency:** By making an optimistic assumption about output tokens, it allows for maximum utilization of the LLM's capabilities without overcomplicating the control mechanism.
- **Flexibility:** The formulas can be easily adjusted if LLM token limits change in the future.

#### Tradeoff

The tradeoff here is that the higher the rate limit, the smaller the max input token size and bigger the output reserve limit which will not be fully utilized

This approach strikes a balance between the need for token management and the desire for a straightforward, maintainable system. It leverages existing API gateway capabilities while providing a clear method for controlling input size, resulting in a robust yet uncomplicated solution for Generative AI API management.

> In the python notebook, `api-limits-design.ipynb`, I have implented these function and tabulated different constraints for given popular models if they are server by either API Gateway of a Queue polling mechanism (the idea remains the same)

