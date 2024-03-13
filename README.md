# Modallama: Modal conversations for interactive, multi-task agents

## Why Modallama?

Modallama helps organize your application's LLM interactions by providing conversational "modes".
Similar to the way that routes organize the user's experience of your UI, modes organize the user's experience of your AI agent.

By changing modes you can change:
* The context and message history provided to the LLM
* The system prompt provided to the LLM
* The tools available to the LLM
* The content and appearance of the UI

Modes allow the LLM to focus on a specific task or outcome, and dynamically change focus to different tasks as appropriate.
Modallama enables rich application experiences powered by LLMs.
Encapsulating the LLM's behaviors, makes these rich applications easier to write, test and re-use.

## Design principles

* Do one thing well: modal conversations with an LLM
	* Creating a mode creates a new AI state.
	* Each mode builds its own state and tools.
	* Modes may be actived by application code (with `setMode`) or the LLM (with tools).

* Play nicely with existing tools
	* Vercel's [`ai/rsc`](https://sdk.vercel.ai/docs/concepts/ai-rsc) library
	* NextJS / React routers
	* State management like Zustand, Redux, etc

* Minimal differences from the `ai/rsc` interface
	* Initial state created on mode entry rather than on `createAI`, since each mode has its own state.
	* Modes define how to `render`, since different modes may render differently.
	* Initial mode provided to `createModalAI`, since we need to start somewhere.

## Example 

### Define modes

First, we'll teach the LLM how to book flights for customers using several tools (defined elsewhere).
Each tool determines how to render the UI when it's used.

```typescript
const bookFlight = mode({
	// Configure the model used for this mode - different modes may use different models as appropriate
	model: {
		provider: openai,
		name: 'gpt-4-turbo'
        },

	// This string describes the mode's purpose and behavior to the LLM, 
	// and is used by the LLM to decide if/when to enter this mode
	description: "Book a flight",

	// The mode's parameters - set by the LLM when the mode begins
	parameters: z.object({
		request: z.string().describe("Information about the flight to book"),
	}),

	// The mode's initial state, given the call parameters.
	initialAIState: ({ request }) {
		role: 'user' | 'assistant' | 'system' | 'function';
		content: string;
		id?: string;
		name?: string;
	}[] = [
		{ role: 'system', content: `
			You are a helpful flight-booking assistant.

			Your goal is to help the user buy a ticket:
			
			1. Gather information from the user about when and where they wish to fly.
			2. Find flights that match their requirements.
			3. Confirm that the user would like to purchase the selected flights.
			4. Buy the ticket.
			
			Use the provided tools to complete this process.`},
		{ role: 'user', request},
	],

	// Tools available to the LLM.
	tools: [
		searchFlights,			// Use the airline's API to search for flights
		confirmPurchase			// Let the user to confirm before charging them
		buyTicket,				// Use the airline's API to purchase a ticket
	],

	// Render non-tool LLM responses (tools control their own rendering)
	text: ({ content }) => <p>{content}</p>,
})
```

Next, we'll teach the LLM how to answer questions about the airline.
Notice that we've changed the system prompt, constructed a new AI state, and provided different tools than before.


```typescript
const policyQA = mode({
	// Configure the model used for this mode - different modes may use different models as appropriate
	model: {
		provider: openai,
		name: 'gpt-4-turbo'
	},

	// This string describes the mode's purpose and behavior to the LLM, 
	// and is used by the LLM to decide if/when to enter this mode
	description: "Answer questions about airline policies",

	// The mode's parameters - set by the LLM when the mode begins
	parameters: z.object({
		question: z.string().describe("A question to answer"),
	}),

	// The mode's initial state, given the call parameters.
	initialAIState: ({ request }) {
		role: 'user' | 'assistant' | 'system' | 'function';
		content: string;
		id?: string;
		name?: string;
	}[] = [
		{ role: 'system', content: `
			You are a helpful airline policy assistant.

			Your goal is to answer any questions the user asks based on information 
			in the company knowledge base.
			
			ONLY answer questions based on information in the knowledge base.
			If you can't answer based on information in the knowledge base tell the 
			user you don't know.`}
		{ role: 'user', request},
	],

	// Tools available to the LLM.
	tools: [
		searchKnowledgeBase,	// RAG search of the company KB
	],

	// Render non-tool LLM responses (tools control their own rendering)
	text: ({ content }) => <p>{content}</p>,
})
```

Finally, we'll create an orientation mode.

The LLM may choose to enter another mode based on the user input, for example, if the user message is "I'd like to book a flight to Hawaii", the LLM should return a tool invocation like `bookFlight(destination: "Hawaii")`, which causes the flight-booking mode to be activated.

```typescript
const orientation = mode({
	// Configure the model used for this mode - different modes may use different models as appropriate
	model: {
		provider: openai,
		name: 'gpt-4-turbo'
	},

	// This string describes the mode's purpose and behavior to the LLM, 
	// and is used by the LLM to decide if/when to enter this mode
	description: "Introduce this application's capabilities and route the user to the next step",

	// The mode's initial state, given the call parameters.
	initialAIState: ({ request }) {
		role: 'user' | 'assistant' | 'system' | 'function';
		content: string;
		id?: string;
		name?: string;
	}[] = [
		{ role: 'system', content: `
			You are a helpful airline assistant.

			Your goal is to help the user solve problems in any way you can.
			
			If the user needs help, explain the different ways you can help, 
			based on the tools available to you.`}
		{ role: 'user', request},
	],

	// Tools available to the LLM.
	tools: [
		bookFlight,
		policyQA,
	],

	// Render non-tool LLM responses (tools control their own rendering)
	text: ({ content }) => <p>{content}</p>,
})
```

### Create server-side AI

```typescript
// Handle messages sent by the user from the client.
async function submitUserMessage(userInput: string) {
	'use server';

	const { renderMode } = useMode<typeof AI>();
	const modeState = getMutableAIState<typeof AI>();

	modeState.update([
		...modeState.get(),
		{
			role: 'user',
			content: userInput,
		},
	]);

	// Call the LLM as configured by the current mode.
	const ui = renderMode();

	return {
		id: Date.now(),
		display: ui
	};
}

// Change modes and begin the flight booking process.
async function bookFlightAction(request: string) {
	'use server';	

	const { renderMode, setMode } = useMode<typeof AI>();

	// Change modes, and provide the new mode's creation arguments.
	setMode(bookFlight, {request: string});

	// Call the LLM using the new mode.
	const ui = renderMode();

	return {
		id: Date.now(),
		display: ui
	};
}

// The initial UI state.
// TODO: Change this to use event sourcing with Zustand or similar
const initialUIState: {
  id: number;
  display: React.ReactNode;
}[] = [];

// Create the AI object. 
// From here everything is the same as when using 
// https://sdk.vercel.ai/docs/concepts/ai-rsc
export const AI = createModalAI({
	actions: {
		submitUserMessage,
		bookFlightAction,
	},
	initialMode: orientation,
	initialUIState,
})
```
