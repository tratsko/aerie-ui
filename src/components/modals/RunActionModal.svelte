<svelte:options immutable={true} />

<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  import { userSequences } from '../../stores/sequencing';
  import type { ActionDefinition, ActionParametersMap } from '../../types/actions';
  import type { User } from '../../types/app';
  import type { ArgumentsMap, FormParameter } from '../../types/parameter';
  import { getUserSequencesInWorkspace, valueSchemaRecordToParametersMap } from '../../utilities/actions';
  import effects from '../../utilities/effects';
  import { getArguments, getFormParameters } from '../../utilities/parameters';
  import Parameters from '../parameters/Parameters.svelte';
  import Modal from './Modal.svelte';
  import ModalContent from './ModalContent.svelte';
  import ModalFooter from './ModalFooter.svelte';
  import ModalHeader from './ModalHeader.svelte';

  export let actionDefinition: ActionDefinition;
  export let parameters: ArgumentsMap | undefined;
  export let user: User | null;

  let running: boolean = false;
  let argumentsMap: ArgumentsMap = {};
  let parametersMap: ActionParametersMap = {};
  let secretParametersMap: ActionParametersMap = {};

  const dispatch = createEventDispatcher<{
    close: void;
    complete: { actionRunId: number | null };
  }>();

  $: if (parameters !== undefined) {
    argumentsMap = parameters;
  }

  $: {
    parametersMap = valueSchemaRecordToParametersMap(actionDefinition.parameter_schema);
  }

  async function run() {
    running = true;

    // Filter out the secret params to send directly to the action server.
    for (const param of Object.keys(parametersMap)) {
      if (parametersMap[param].schema.type === 'secret') {
        secretParametersMap[param] = argumentsMap[param];
        delete parametersMap[param];
        delete argumentsMap[param];
      }
    }

    const actionRunId = await effects.createActionRun(
      actionDefinition.id,
      argumentsMap,
      actionDefinition.settings,
      Object.keys(secretParametersMap).length > 0, // The DB only needs to know if there are secrets or not.
      user,
    );

    if (actionRunId !== null) {
      await effects.sendActionSecretParameters(secretParametersMap, actionRunId, user);
    }

    running = false;
    dispatch('complete', { actionRunId });
  }

  function onChangeFormParameters(event: CustomEvent<FormParameter>) {
    const { detail: formParameter } = event;
    if (formParameter.schema.type === 'options-single') {
      const sequences = $userSequences.find(sequence => sequence.id === parseInt(formParameter.value));
      formParameter.value = sequences?.name ?? null;
      argumentsMap = getArguments(argumentsMap, formParameter);
    } else if (formParameter.schema.type === 'options-multiple') {
      const ids: string[] = formParameter.value;
      let sequenceNames: string[] = [];
      ids.forEach(id => {
        const seq = $userSequences.find(sequence => sequence.id === parseInt(id));
        if (seq !== undefined) {
          sequenceNames.push(seq.name);
        }
      });
      formParameter.value = sequenceNames;
      argumentsMap = getArguments(argumentsMap, formParameter);
    } else {
      argumentsMap = getArguments(argumentsMap, formParameter);
    }
  }
</script>

<Modal height="max-content" width={500}>
  <ModalHeader on:close>Run Action</ModalHeader>

  <ModalContent style="max-height: 50vh;overflow: auto">
    <div class="st-typography-label pb-2">Input parameters to run <b>{actionDefinition.name}</b></div>
    <Parameters
      formParameters={getFormParameters(
        parametersMap,
        argumentsMap,
        [],
        undefined,
        undefined,
        getUserSequencesInWorkspace($userSequences, actionDefinition.workspace_id),
        'sequence',
      )}
      parameterType="action"
      hideRightAdornments
      hideInfo
      on:change={onChangeFormParameters}
    />
  </ModalContent>

  <ModalFooter>
    <button class="st-button secondary" on:click={() => dispatch('close')}> Cancel </button>
    <button class="st-button" disabled={running} on:click={run}>
      {running ? 'Running...' : 'Run'}
    </button>
  </ModalFooter>
</Modal>
