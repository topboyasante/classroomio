<script lang="ts">
  import Modal from '$lib/components/Modal/index.svelte';
  import TextField from '$lib/components/Form/TextField.svelte';
  import PrimaryButton from '$lib/components/PrimaryButton/index.svelte';
  import CheckmarkFilledIcon from 'carbon-icons-svelte/lib/CheckmarkFilled.svelte';
  import CheckmarkOutlineIcon from 'carbon-icons-svelte/lib/CheckmarkOutline.svelte';
  import ComingSoon from '$lib/components/ComingSoon/index.svelte';
  import { Tag } from 'carbon-components-svelte';
  import { TEMPLATES, TAGS } from '$lib/mocks';
  import type { ExerciseTemplate } from '$lib/utils/types';
  import { lesson } from '../store/lessons';
  import { useCompletion } from 'ai/svelte';
  import Confetti from '$lib/components/Confetti/index.svelte';
  import { toggleConfetti } from '$lib/components/Confetti/store';
  import { VARIANTS } from '$lib/components/PrimaryButton/constants';
  import { getTextFromHTML } from '$lib/utils/functions/course';
  import { writable } from 'svelte/store';
  import { Circle3 } from 'svelte-loading-spinners';

  export let open = false;
  export let handleAddExercise = () => {};
  export let handleCancelAddExercise = () => {};
  export let handleTemplateCreate: (template: ExerciseTemplate) => Promise<void>;
  export let title = '';

  enum Type {
    SCRATCH = 0,
    TEMPLATE = 1,
    AI = 2
  }

  let step = 0;
  let type: Type = Type.SCRATCH;
  let questionNumber = 5;
  let optionNumber = 5;
  let isLoading = writable(false);
  let isAIStarted = false;
  let note = '';

  const options = [
    {
      title: 'From Scratch',
      subtitle: 'Create your exercise from scratch if you have your questions already prepared',
      type: Type.SCRATCH,
      isDisabled: false
    },
    {
      title: 'Use a Template',
      subtitle: 'Select from 100+ templates of predefined exercise to help you get started',
      type: Type.TEMPLATE,
      isDisabled: false
    },
    {
      title: 'Use AI',
      subtitle: 'You can generate an exercise with AI from your notes',
      type: Type.AI,
      isDisabled: false
    }
  ];

  const tags = Object.values(TAGS);
  let selectedTag = tags[0];
  let selectedTemplateId = '';
  let isTemplateFinishedLoading = false;

  const { input, handleSubmit, completion } = useCompletion({
    api: '/api/completion/exerciseprompt',
    onFinish: async (prompt: string, completion: string) => {
      if (!$lesson.id) return;

      toggleConfetti();
      const template: ExerciseTemplate = JSON.parse($completion);
      await handleTemplateCreate(template);
      toggleConfetti();
      $isLoading = false;
    }
  });

  function handleNext() {
    step = step + 1;
  }

  function handleBack() {
    step = step - 1;
  }

  function callAI() {
    $input = JSON.stringify({
      questionNumber,
      optionNumber,
      lessonNote: note
    });

    setTimeout(() => {
      isAIStarted = true;
      $isLoading = true;
      handleSubmit({ preventDefault: () => {} });
    }, 500);
  }

  $: note = getTextFromHTML($lesson?.materials?.note || '');
</script>

<Modal
  onClose={handleCancelAddExercise}
  bind:open
  modalHeading="New Exercise"
  maxWidth="max-w-4xl"
  width="w-4/5"
>
  {#if !$isLoading && isAIStarted}
    <Confetti />
  {/if}
  {#if step === 0}
    <div>
      <h2 class="text-2xl font-medium my-5">How do you want to create your exercise?</h2>

      <div class="flex gap-2 justify-between my-8">
        {#each options as option}
          <button
            class="w-[261px] h-[240px] p-5 rounded-md dark:bg-neutral-700 border-2 {option.type ===
            type
              ? 'border-primary-400'
              : `border-gray-200 dark:border-gray-400 ${
                  !option.isDisabled && 'hover:scale-95'
                }`} flex flex-col {option.isDisabled &&
              'cursor-not-allowed opacity-60'} transition-all ease-in-out"
            type="button"
            on:click={!option.isDisabled ? () => (type = option.type) : undefined}
          >
            <div class="w-full flex flex-row-reverse h-[70%]">
              {#if option.type === type}
                <CheckmarkFilledIcon
                  size={16}
                  class="carbon-icon text-primary-600 dark:text-primary-200"
                />
              {:else if !option.isDisabled}
                <CheckmarkOutlineIcon size={16} class="carbon-icon" />
              {/if}
            </div>

            <div>
              <p class="font-bold text-start flex items-center">
                <span class="mr-2 text-sm">{option.title}</span>
                {#if option.isDisabled}
                  <ComingSoon />
                {/if}
              </p>
              <p class="text-xs font-light text-start">{option.subtitle}</p>
            </div>
          </button>
        {/each}
      </div>

      <div class="mt-8 flex items-center flex-row-reverse">
        <PrimaryButton className="px-6 py-3" label="Next" onClick={handleNext} />
      </div>
    </div>
  {:else if step === 1}
    {#if type === Type.SCRATCH}
      <div class="flex items-center justify-center w-96 m-auto min-h-[300px]">
        <div class="w-full">
          <h2 class="text-2xl font-medium my-5">Give your exercise a title</h2>
          <TextField
            bind:value={title}
            autoFocus={true}
            placeholder="Exercise name"
            className="my-4"
          />

          <div class="mt-5 flex items-center justify-between">
            <PrimaryButton
              className="px-6 py-3"
              label="Back"
              variant={VARIANTS.OUTLINED}
              onClick={handleBack}
            />
            <PrimaryButton className="px-6 py-3" label="Finish" onClick={handleAddExercise} />
          </div>
        </div>
      </div>
    {:else if type === Type.TEMPLATE}
      <div>
        <h2 class="text-2xl font-medium my-5">Select a template</h2>

        <div>
          <div class="mb-5 flex items-center gap-2">
            {#each tags as tag}
              <Tag
                type={selectedTag === tag ? 'warm-gray' : 'outline'}
                class={selectedTag === tag ? 'bg-primary-400' : ''}
                interactive
                on:click={() => {
                  selectedTag = tag;
                  selectedTemplateId = '';
                }}>{tag}</Tag
              >
            {/each}
          </div>

          <div class="flex flex-wrap items-start gap-2">
            {#each TEMPLATES[selectedTag] as template}
              <button
                class="w-[161px] h-[140px] hover:scale-95 p-5 rounded-md dark:bg-neutral-700 border-2 {template.id ===
                selectedTemplateId
                  ? 'border-primary-400'
                  : `border-gray-200 dark:border-gray-400`} flex flex-col transition-all ease-in-out"
                type="button"
                on:click={() => (selectedTemplateId = template.id)}
              >
                <div class="flex flex-col justify-evenly h-full">
                  <p class="font-bold text-sm text-start flex items-center">
                    {template.title}
                  </p>
                  <div class="flex flex-col items-start justify-between gap-1">
                    <p class="text-xs font-normal">
                      {template.questions} Questions
                    </p>
                    <p class="text-xs font-normal text-start">
                      {template.points} Points
                    </p>
                  </div>
                </div>
              </button>
            {/each}
          </div>

          <div class="mt-5 flex items-center justify-between">
            <PrimaryButton
              className="px-6 py-3"
              label="Back"
              variant={VARIANTS.OUTLINED}
              onClick={handleBack}
            />
            <PrimaryButton
              isDisabled={!selectedTemplateId}
              className="px-6 py-3"
              label="Finish"
              isLoading={isTemplateFinishedLoading}
              onClick={async () => {
                isTemplateFinishedLoading = true;
                const template = TEMPLATES[selectedTag].find((t) => t.id === selectedTemplateId);
                if (!template) return;

                console.log('Selected template', template);
                await handleTemplateCreate(template.data);
                isTemplateFinishedLoading = true;
              }}
            />
          </div>
        </div>
      </div>
    {:else if type === Type.AI}
      <div>
        <div class="flex flex-row justify-between max-h-[500px]">
          <div class="w-[60%] mr-1 border px-3 py-2 rounded-md">
            {#if note.length}
              <h3>Create exercises from Notes with AI</h3>
              <p class="text-sm mb-4">
                Choose how many questions and options you want and AI will help you create an
                exercise out of your note. Let's go.
              </p>
              <TextField
                label="How many questions do you want to create?"
                type="number"
                bind:value={questionNumber}
                placeholder="5"
                className="mb-2"
                isRequired
              />
              <TextField
                label="How many options per questions do you want?"
                type="number"
                bind:value={optionNumber}
                placeholder="5"
                isRequired
              />
            {:else}
              <h3>Please add a note to use this feature</h3>
            {/if}
            <div class="mt-5 flex items-center flex-row-reverse">
              <PrimaryButton
                onClick={callAI}
                isLoading={$isLoading}
                isDisabled={$isLoading || !note}
                variant={VARIANTS.OUTLINED}
              >
                Generate
              </PrimaryButton>
            </div>
          </div>
          <div
            class="w-[40%] px-5 py-3 border rounded-md overflow-y-auto flex justify-center items-center"
          >
            {#if $isLoading}
              <Circle3 size="60" unit="px" duration="1s" />
            {:else if isAIStarted}
              <p class="max-h-[200px] leading-7 text-sm">AI Generation Complete</p>
            {:else}
              <p class="max-h-[200px] leading-7 text-sm">Click "Generate" for some magic</p>
            {/if}
          </div>
        </div>
        <div class="mt-5 flex items-center justify-between">
          <PrimaryButton
            className="px-6 py-3"
            label="Back"
            variant={VARIANTS.TEXT}
            onClick={handleBack}
          />
          <PrimaryButton className="px-6 py-3" label="Finish" onClick={handleAddExercise} />
        </div>
      </div>
    {/if}
  {/if}
</Modal>
