<script lang="ts">
	import { onMount, tick } from 'svelte';
	import { browser } from '$app/environment';

	import {
		Title,
		Group,
		Stack,
		Card,
		Button,
		Progress,
		ActionIcon,
		Text,
		Modal
	} from '@svelteuidev/core';
	import { Confetti } from 'svelte-confetti';
	import { allGradeWords } from '$lib/words';
	import type * as types from '$lib/types';

	let word = '';
	let value = '';
	let progress = 0;
	let gradeIndex = 0;
	const gradeLevelNames = Object.keys(allGradeWords);
	let gradeLevel;
	let gradeWords;
	let spellingError = false;
	let showLevelUpConfetti = false;

	let numberOfTries = 0;
	let showMessage = false;
	let showWinnerMessage = false;

	let inputElement;

	// Word Definition and Examples
	let wordDefinition: types.WordDefinition;
	let examplePartOfSpeechIndex = 0;
	let exampleIndex = 0;

	// Audio Word
	let synthesis;
	let audio: HTMLAudioElement;
	let voices;
	let voiceIndex = 0;

	// Constants
	const progressIncrement = 10;

	onMount(() => {
		audio = new Audio();
		synthesis = window.speechSynthesis;
		voices = synthesis.getVoices().filter((voice) => voice.lang === 'en-US');

		inputElement.focus();
		gradeIndex = window.localStorage.getItem('gradeIndex')
			? Number(window.localStorage.getItem('gradeIndex'))
			: 0;
		gradeLevel = gradeLevelNames[gradeIndex];
		gradeWords = allGradeWords[gradeLevel];
		nextWord();
	});

	function _nextVoice() {
		voiceIndex = voiceIndex + 1 < voices.length ? voiceIndex + 1 : 0;
	}

	function _audioWord() {
		let url = '';
		wordDefinition.phonetics.forEach((phonetic) => {
			if (phonetic.audio !== '') {
				url = phonetic.audio;
			}
		});
		return url;
	}

	function _nextMeaningExample() {
		if (exampleIndex + 1 < wordDefinition.meanings[examplePartOfSpeechIndex].definitions.length) {
			exampleIndex = exampleIndex + 1;
		} else {
			// all of the examples from the current part of speech have been shown. Going to the next part of speech
			exampleIndex = 0;
			examplePartOfSpeechIndex =
				examplePartOfSpeechIndex + 1 < wordDefinition.meanings.length
					? examplePartOfSpeechIndex + 1
					: 0;
		}
	}

	async function nextWord() {
		examplePartOfSpeechIndex = 0;
		exampleIndex = 0;
		numberOfTries = 0;
		spellingError = false;
		word = gradeWords[Math.floor(Math.random() * gradeWords.length)];
		value = '';
		readWord();
	}

	function nextGrade() {
		if (gradeIndex === gradeLevelNames.length - 1) {
			showWinnerMessage = true;
			console.log('showWinnerMessage', showWinnerMessage);
			return;
		}
		showLevelUpConfetti = true;
		numberOfTries = 0;
		progress = 0;
		gradeIndex = (gradeIndex + 1) % gradeLevelNames.length;
		gradeLevel = gradeLevelNames[gradeIndex];
		gradeWords = allGradeWords[gradeLevel];
		window.localStorage.setItem('gradeIndex', gradeIndex.toString());
		nextWord();
	}

	async function readWord() {
		if (!wordDefinition || wordDefinition.word !== word) {
			await getWordDefinition();
		}
		let audioUrl = _audioWord();
		if (audioUrl) {
			audio.src = audioUrl;
			audio.play();
		} else {
			let utterance = new SpeechSynthesisUtterance(word);
			utterance.voice = voices[voiceIndex];
			synthesis.speak(utterance);
			_nextVoice();
		}
	}

	async function getWordDefinition() {
		let url = `https://api.dictionaryapi.dev/api/v2/entries/en/${word}`;
		let response = await fetch(url);
		if (response.ok) {
			let json = await response.json();
			wordDefinition = json[0];
		} else {
			console.log('HTTP-Error: ' + response.status);
			wordDefinition = {
				word: word,
				phonetics: [],
				meanings: [],
				license: {
					name: '',
					url: ''
				},
				sourceUrls: []
			};
		}
	}

	async function readWordInASentence() {
		if (!wordDefinition || wordDefinition.word !== word) {
			await getWordDefinition();
		}
		let textToRead = `
        Part of speech: ${wordDefinition.meanings[examplePartOfSpeechIndex].partOfSpeech}.
        Word definition: ${
					wordDefinition.meanings[examplePartOfSpeechIndex].definitions[exampleIndex].definition ??
					`Sorry. I do not have a good definition for ${word}.`
				}.
        ${
					wordDefinition.meanings[examplePartOfSpeechIndex].definitions[exampleIndex].example
						? 'Used in a sentence:' +
						  wordDefinition.meanings[examplePartOfSpeechIndex].definitions[exampleIndex].example +
						  '.'
						: ''
				}`;
		let utterance = new SpeechSynthesisUtterance(textToRead);
		utterance.voice = voices[voiceIndex];
		synthesis.speak(utterance);
		_nextVoice();
		_nextMeaningExample();
	}

	function spellingCorrect() {
		if (value.toLowerCase() === word.toLowerCase()) {
			spellingError = false;
			return true;
		} else {
			spellingError = true;
			return false;
		}
	}

	function checkSpelling() {
		numberOfTries += 1;
		if (spellingCorrect()) {
			progress += progressIncrement;
			value = '';
			if (progress >= 100) {
				nextGrade();
			} else {
				nextWord();
			}
		} else {
			if (numberOfTries >= 3) {
				showMessage = true;
				nextWord();
			}
			spellingError = true;
			progress -= progressIncrement;
			if (progress < 0) {
				progress = 0;
			}
		}
		inputElement.focus();
	}

	function restartAtBeginning() {
		showWinnerMessage = false;
		gradeIndex = 0;
		gradeLevel = gradeLevelNames[gradeIndex];
		gradeWords = allGradeWords[gradeLevel];
		nextWord();
		window.localStorage.setItem('gradeIndex', gradeIndex.toString());
		window.location.reload();
	}

	$: console.log(word);
	$: console.log(wordDefinition);
</script>

<svelte:head>
	<title>Home</title>
	<meta name="description" content="Simple Spelling Bee App" />
</svelte:head>

<section>
	{#key showLevelUpConfetti}
		{#if showLevelUpConfetti}
			<div class="confetti">
				<Confetti
					x={[-5, 5]}
					y={[0, 0.1]}
					delay={[0, 2000]}
					duration={5000}
					amount={700}
					fallDistance="100vh"
				/>
			</div>
		{/if}
	{/key}

	<Modal opened={showWinnerMessage} on:close={restartAtBeginning} title="Winner!">
		<Text>
			You have completed all the words in the app. You are a spelling bee champion! The app will now
			reset back to first grade.
		</Text>
	</Modal>

	<Card p="lg" override={{ marginTop: 20 }}>
		<Stack>
			<Title
				variant="gradient"
				gradient={{ from: 'blue', to: 'red', deg: 45 }}
				override={{ lineHeight: '2em' }}>Spelling Bee</Title
			>

			<Progress radius="xl" size="xl" value={progress} label={`${progress}%`} />

			<Title
				order={2}
				variant="gradient"
				gradient={{ from: 'blue', to: 'red', deg: 45 }}
				override={{ lineHeight: '2em' }}
			>
				Current word level: {gradeLevel}
			</Title>

			<Group grow>
				<Button variant="outline" on:click={readWord} color="black">
					Hear Word
					<span style="transform: rotate(-180deg); margin-left: 10px; margin-top:3px">🕪</span>
				</Button>
				<Button variant="outline" on:click={readWordInASentence} color="black">
					Define word <span style="transform: rotate(-180deg); margin-left: 10px; margin-top:3px"
						>🕪</span
					></Button
				>
			</Group>

			<Group>
				<div />
				<input
					on:focus={() => (spellingError = false)}
					bind:value
					bind:this={inputElement}
					placeholder="Enter spelling word"
					class:error={spellingError}
				/>

				<Button variant="outline" on:click={checkSpelling} override={{ height: 51 }}>Check</Button>
			</Group>

			{#if numberOfTries == 0}
				<Text variant="gradient" override={{ lineHeight: '1.5em' }}>
					Enter the spelling word and click check
				</Text>
			{:else if numberOfTries == 1}
				<Text variant="gradient" override={{ lineHeight: '1.5em' }}>Give it one more shot</Text>
			{:else}
				<Text variant="gradient" override={{ lineHeight: '1.5em' }}>The word is {word}</Text>
			{/if}
		</Stack>
	</Card>
</section>

<style>
	section {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		flex: 0.6;
		max-width: 500px;
		margin-left: auto;
		margin-right: auto;
	}

	.confetti {
		position: fixed;
		top: -50px;
		left: 0;
		z-index: 100;
		height: 100vh;
		width: 100vw;
		display: flex;
		justify-content: center;
		overflow: hidden;
		pointer-events: none;
	}

	::placeholder {
		color: lightgray;
	}

	input {
		color: black;
		width: 350px;
		border: 0px;
		border-bottom: 2px solid lightgray;
		font-size: x-large;
		padding: 12px;
	}

	input:focus {
		outline: none;
		transition: border-color 200ms ease 0s;
		transition-duration: 200ms;
		transition-timing-function: ease;
		transition-delay: 0s;
		transition-property: border-color;
		border-bottom: 2px solid #228be6;
	}

	.error {
		animation: shake 0.82s cubic-bezier(0.36, 0.07, 0.19, 0.97) both;
		transform: translate3d(0, 0, 0);
		backface-visibility: hidden;
		perspective: 1000px;

		color: red;
		border-bottom-color: red;
		backface-visibility: hidden;
	}

	@keyframes shake {
		10%,
		90% {
			transform: translate3d(-1px, 0, 0);
		}

		20%,
		80% {
			transform: translate3d(2px, 0, 0);
		}

		30%,
		50%,
		70% {
			transform: translate3d(-4px, 0, 0);
		}

		40%,
		60% {
			transform: translate3d(4px, 0, 0);
		}
	}
</style>
