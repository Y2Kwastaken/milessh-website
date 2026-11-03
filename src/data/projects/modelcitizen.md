---
title: 'Model Citizen'
tech: ['Go 1.27.1', 'GRPC', 'Disgo', 'Docker', 'Claude Code']
lead: 'A fun and customziable discord bot assistant template that can be deployed with ease via docker compose.'
---

This is an informal writeup on this project, regarding the things I enjoyed, things I didn't and how I worked through this project.
I wouldn't consider this project totally done, but I think I've lost the extreme interest I've had for now. As such this article is only
accurate as of writing on 11/02/2026.

Note: This is my first go project in a year or two, so I was also learning golang in this process.

## Features

Model Citizen features are loosely as follows:

- Discord Music Bot,
- Voice Activated Music Commands,
- Voice "Assistant" with customizable personalities, and minimal memory capabilities

## Version 1.0 An Blunt Autopsy

The [1.0 version of Model Citizen](https://github.com/Y2Kwastaken/model-citizen/tree/v1) was an informal and infantile version of the current
Model Citizen. After scaling to all of the features I want I felt extremely dissatisifed with this project. I realized, I don't understand it at all.
This honestly gives me great issues considering the features list above hinges mostly on music functionality. I honestly struggled with this for a while
and felt like a rewrite was in order. I think this project was the first where I "vibe coded" the music portion of the project very heavily.

### Self Reflection On AI Use

I felt In order for a rewrite to succeed it would be necessary for me to succinctly identify points of issue on AI use that lead to me to this outcome.
Honestly, it lead me to quite the scary rabbit hole. I ought to say AI use in this project lead me to discard parts of programming, which, I would consider fun and a hobby. There was absolutely no struggle to implement any of the music portion of this project. It made me further consider what is AI doing to [my brain?](https://www.apa.org/monitor/2026/09/ai-brain-development-youth) The true answer is probably still unknown to a degree, but I felt that I was becoming, complacent, dumb, unwilling to learn. The irony of this is not lost on a website that has it's CSS and some components completely coded by AI. Either way I realized I would need to minimize or cut out how I use AI in the rewrite.

### Design Errors

While AI wrote much of the core code loop, I ultimately architected V1 and there were many design flaws. For one the AI component despite being written generically was very dependent on the discord portion. To me this typically signals as an issue. Two distinct components have succinct components, which are inherently tied together. Let us inspect the LanguageModel interface

```go
type Origin struct {
	Guild   snowflake.ID
	Channel snowflake.ID
	Caller  snowflake.ID
}


type LanguageModel interface {
	HasFeature(feature ModelFeature) bool
	MemorySet() MemorySet
	Tools() ModelTools
	History() HistoryProvider

	// tweaks
	SetPersonality(guild snowflake.ID, name string) error

	// functions
	Chat(ctx context.Context, origin Origin) (string, error)
	Transcribe(ctx context.Context, clip Clip) (string, error)
	Speak(ctx context.Context, text string) (Clip, error)
}
```

If we look at the interface here we can see a couple issues in my eyes.

1. History and Memory are separate ideas
2. LanguageModel encapsulates to many functionalities
3. snowflake.ID and Origin ties this to discord specifically

#### History and Memory

Thinking about this more deeply what is memory? Memory is a set of user messages that allows the bot to build a history about specific users and conversations over a longer period of a time (While not quite being long term). History is a specific rolling iteration of the last `n` messages. This allows the bot to build a conversation history and reply in a relevant fashion.  

Note in both of these explanations we see that history and memory share that they build context. Further we actually might be able to extend this logic of "Memory" to encapsulate "Personalities" and we will take a look at how I handle that in v2.

#### LanguageModel Interface Design

The interface design of LanguageModel is just too "chunky" by this I mean it carries around unrelated dead weight. Think about speaking and transcribing. Both speaking and transcribing require nothing related to the functionalities of LanguageModel's themselves. Language Model's can open up a "Chat" no matter the context. This leads to a situation like this:


```go
go func() {
	defer wg.Done()
	text, err := ears.brain.Transcribe(ctx, model.Clip{Data: audio.EncodeWAV(u.PCM), Format: "wav", Language: hearing.Language})
	if err != nil {
		slog.Error("transcribing utterance", slog.String("user_id", u.User.String()), slog.Any("err", err))
		return
	}
	texts[i] = text
}()
```

Within this code I transcriber the utterances and append them to an array, but stop to a minute to consider? What part of this requires any of the benefits the
LanguageModel provides. None of the inner systems are used and it is distinct from chatting, there is no response, or articulation, this is all delegated to chat later and appended as memory. This means we could easily cut out Transcribe and Speak from the interface they are there for perceived "convenience." In my mind when we get to v2 these should be separate ideas.

#### Discord Ties

To be honest... This category can kind of go under explained, for a sufficiently generic interface like this, there should be no reason we need to tie ourselves to discord. Disgo's snowflake/v2 is sufficiently small that I could depend on it in my LLM module, but I see no reason for this dependency when it can easily be expressed by a long value.


### Grandiose Idea of Separation

To be honest part of this idea was also driven by a more grandiose idea of separation one in which the llm component is not just a separate module, but a separate service. I decided relatively early before the redesign that there would be advantages to a generic GRPC service. This includes 1 grpc service possibly serving multiple bots. *I didn't quite get to such an idea coming to fruition, though, I'm okay with that.*

## Version 2.0 Learning Lessons

When it came to implementing V2 I changed the project layout further. there are instead 3 modules `discord-bot`, `llm` and `shared`. All of these components have
their own self contained projects. Shared houses generated GRPC code and shared utilities, this one is a no brainer required for me. Next the LLM module, learning about Go generics allowed me to leverage generic data structures that were repeated in my code in 1.0 mainly the circular queue. Second I completely severed any connection between disgo and my llm module, which gave me the disconnection I desired.

### The New LanguageModel

```go
type LanguageModel interface {
	// Features
	HasFeature(flag FeatureFlag) bool
	Features() []FeatureFlag
	// Tools
	ToolSet() (tools.ToolSet, error)
	// Memory
	Memory(id int64, sid string) (memory.MemoryHolder, bool)
	AddMemoryHolder(id int64, sid string, holder memory.MemoryHolder) bool
	SystemMemory(id int64, sid string) bool
	// the sid of the system memory set for id, false if none has been set
	SystemMemoryName(id int64) (string, bool)
	// Actions
	Chat(ctx context.Context, source util.Source, incoming []memory.Memory) (string, error)
	Act(ctx context.Context, source util.Source, request string) ([]memory.ToolCall, error)
}
```

As you see the new LanguageModel actually addresses a lot of the issues I had. We have a new feature "Act" which acts on an action without any bias on current conversations. I found that specific iterations of personalities were very reluctant to run commands, so handing commands off to an unbiased actor, while slow, gives a better objective outcome for those who just want to utilize commands and move on. This also addresses my issues with Memory, which I'll go more into depth on next.

### Memory

Next the consolidation of memory, history, and system memory to allow for a streamlined process. This was to an inherent trade-off, however. 

```go
type Identity int
const (
	NONE Identity = iota
	SELF
	OTHER
)
type ToolCall struct {
	ID        string
	Name      string
	Arguments string
	Result    string
}
type Memory struct {
	CreatedAt time.Time
	Involves []string
	Content string
	Owner Identity
	ToolCalls []ToolCall
}
type MemoryHolder interface {
	Store(memory Memory) (Memory, bool, error)
	Ordered() []Memory
	Wipe()
}
```

As you can see Memory is a lot more "bloated" now to say the last, but let me explain each part. Many LLM models need to store tool calls to know they've actually committed the call. As such these are only given in "SELF" calls. This bloats the interface, but I believe the limited trade-off is worth it in the long run. This makes memory overall a lot more flexible in it's use cases. Identity exists to allows an LLM to know where to classify it's memory. E.g. other's should probably be classified as regular messages to the LLM but it may be more safe to use these SELF memories as a component of a system prompt. Next the "involves" array tells us who is involved in a conversation. This improves flexibility over v1 because it allows an LLM to assign multiple people to one "memory." Finally it classifies al memories at a time so it would know if something is "old." *Though, this is only more impactful for high parameter models.*

Next the memory holder is very generic interface that gives us all the utilities we could ever want with memory. Why can't we remove specific memories? Well memories have no "identifier" other than time which isn't a good lookup key and any consumer over MemoryHolder should not be concerned with how memories are culled internally on store. Exposing a generic "Remove X memory API" actually seemed more limiting to MemoryHolder in my opinion.

The generic form of memory allows implementations like this to be easily possible.

```go
func (m *brainModel) history(src util.Source) (memory.MemoryHolder, error) {
	if !m.HasFeature(MEMORY) {
		return memory.NewEmptyMemory(), nil
	}

	if holder, ok := m.Memory(src.Id, src.Where); ok {
		return holder, nil
	}

	holder, err := memory.NewShortTermMemory(m.historySize)
	if err != nil {
		return nil, err
	}

	m.AddMemoryHolder(src.Id, src.Where, holder)
	holder, _ = m.Memory(src.Id, src.Where)
	return holder, nil
}
```

Such an implementation allows us to gather a history from a generic source. This could be short-term memory, message history for a specific channel or place,
either way a history is available.

### GRPC, Dictation and Transcription

I'm honestly much less passionate about this section, so I'll keep it brief. GRPC is kind of to me the really boring glue since I've worked with so much server coding I didn't find this intrinsically very interesting. A lot of it was written by AI because GRPC documentation on hooking into modern golang is quite out of date and I was getting quite flustered trying to find modernized sources.

That aside there was seemingly missing components dictation and transcription where did those go to? The answer is honestly relatively boring I just moved them into their respective grpc "servers" 
```go
type speechServer struct {
	pb.UnimplementedSpeechServiceServer
	stt          config.Stt
	transcribers shared.Rotation[clients.Transcriber]
	tts config.Tts
	// voice name to the speakers that can say it
	voices map[string]shared.Rotation[clients.Speaker]
	brain              model.LanguageModel
	defaultPersonality string
	personalities      map[string]config.Personality
}
```

This is a pretty boring but smart solution. Everything self contained in GRPC no one needs to know but the server receiving the audio clips, obfuscate it and box it away for a million years to come.

### Audio Code

A big part of this rewrite was me actually wanting to learn about Audio code. So I set myself aside and did that. I spent a few days learning the intricacies of sampling, mixing, and sampling audio. This was actually quite a fun process and I easily sunk 20 hours into this. I found quite a few very old music resources from the early 2000s for this, but it did allow me to create a system that I could expand on in the future e.g. fade in and out between songs. Pitch changes etc if I ever want to get more into the weeds. What I actually enjoyed about audio code was about how deceptively simple the basics are.

Want to combine 2 sets of audio? I couldn't imagine how hard this was before but to my honest dismay it was literally just adding the two streams together:
```go
func mixFloats(one []float32, two []float32) []float32 {
	len1 := len(one)
	len2 := len(two)

	var arrLength int
	if len1 < len2 {
		arrLength = len2
	} else {
		arrLength = len1
	}

	out := make([]float32, arrLength)
	for i := 0; i < arrLength; i++ {
		var first float32
		if i >= len1 {
			first = 0
		} else {
			first = one[i]
		}

		var second float32
		if i >= len2 {
			second = 0
		} else {
			second = two[i]
		}

		out[i] = first + second
	}

	return out
}
```

Audio programming also allowed me to get in touch with go channels which were extremely informative and useful part of the language. The use of channels here really cemented how much I appreciated go's ease of concurrency. [NextFrames](https://github.com/Y2Kwastaken/model-citizen/blob/main/shared/audio/math.go#L116) method was probably my favorite example of how easy this was made. 

I also created a neat little system to edit specific frames of music. Though in the future It might be very beneficial for me to do more than this:
```go
func ComposeFrameEditors(e1 SampleEditor, e2 SampleEditor) SampleEditor {
	return func(f float32) float32 {
		return e2(e1(f))
	}
}

func GainEditor(db float32) SampleEditor {
	gain := dBToGain(db)
	return func(f float32) float32 {
		return f * gain
	}
}

func RawGainEditor(gain float32) SampleEditor {
	return func(f float32) float32 {
		return f * gain
	}
}
```

I particularly enjoyed the idea of composing multiple frame editors together, I tend to admire the beauty of composition in moments like this.

## Closing Thoughts

Honestly when it came to producing this project I can't say I felt better with the rewrite. I genuinely felt like I left the rewrite learning so many things. I got some audio coding knowledge out of this project and it was beyond satisfying to hear my code come to life when practicing mixing, layering, and applying gain edits etc.

When it comes to AI in my projects it honestly taught me about moderation. There are definitely still AI components in version 2.0, but I tried to keep them separate enough where if I ever do get the passion to learn those parts they should be relatively easy to replace. I'm overall very satisfied with the current state of Model Citizen and if the time comes where I feel like I want to work on it again it feels very approachable to jump back in, especially with the nice little comments I decided to leave myself.

While I feel like I might've skimped out on details at the end this project was truly a joy to work on.