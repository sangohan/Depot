<!--
Copyright 2020 Kyle Kukshtel

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

<script>
import {defaults} from './depotDefaults';
import BooleanField from '../Fields/BooleanField.svelte';
import EnumField from '../Fields/EnumField.svelte';
import LongTextField from '../Fields/LongTextField.svelte';
import NumberField from '../Fields/NumberField.svelte';
import TextField from '../Fields/TextField.svelte';
import { createEventDispatcher } from 'svelte';
import ConfigLineSelect from '../Fields/Config/ConfigLineSelect.svelte';
import ConfigColumnSelect from '../Fields/Config/ConfigColumnSelect.svelte';
const dispatch = createEventDispatcher();
function configUpdate(updateType) {
    dispatch('message', {
        "type" : updateType
    });
}

export let data;
export let config;
export let debug;
let configTitle = "";
$: {
    let first = config.operation === "new" ? "New " : "Edit ";
    let second = config.editType === "sheet" ? "sheet config" : "column config";
    configTitle = first + second;
}
function closeEditor() {
    configUpdate("close");
}
function createBlob() {
    configUpdate("create");
}
function saveBlob() {
    configUpdate("save");
}
function deleteBlob() {
    configUpdate("delete");
}
function upgradeColumn() {
    configUpdate("upgradeColumnData");
}

let validName = true;
// $: disabled = !validName;
$: {
    if(config.active)
    {
        switch (config.editType) {
            case "sheet":
                validName = !config.bannedNames.sheetNames.includes(data.name);
                break;
            default:
                validName = !config.bannedNames.columnNames.includes(data.name);
                break;
        }
    }
}

let columnSettings = {}
$: {
    if(config.operation === "edit")
    {
        columnSettings = data.configurable;
    }
    else if(config.operation === "new")
    {
        columnSettings = defaults[config.editType].configurable;
    }
}

let needsColumnUpdate = false;
$: {
    if(config.operation === "edit" && config.editType !== "sheet") {
        needsColumnUpdate = false;
        Object.keys(defaults[data.typeStr]).forEach(key => {
            if(key !== "configurable") {
                if (!(key in data)){
                    needsColumnUpdate = true;
                    return;
                }
            } else {
            Object.keys(defaults[data.typeStr]["configurable"]).forEach(configKey => {
                if (!(configKey in data["configurable"])){
                    needsColumnUpdate = true;
                    return;
                }
                });
            }
        });
    } else {
        needsColumnUpdate = false;
    }
}

</script>
{#if config.active}
    <p>{configTitle}</p>
    {#if needsColumnUpdate}
        Depot has been updated since you first created this column. Click below to update this column to the latest version.<br>
        Note that any unsaved changes on this column will be reverted upon column update.<br>
        <button on:click={upgradeColumn}>Update Column</button>
        <br>
    {/if}
    <table>
    <tr>
        <td>Field</td>
        <td>Value</td>
    </tr>
    <tr>
        <td>GUID</td>
        <td>{data["guid"]}</td>
    </tr>
    {#each Object.keys(columnSettings) as fieldName}
        {#if typeof columnSettings[fieldName] !== 'object'}
        <tr>
            <td>{fieldName}</td>
            <td>
                <!-- commented out fields aren't used for field config -->
                {#if columnSettings[fieldName] === "text"}
                <div><TextField bind:data={data[fieldName]}/></div>
                {:else if columnSettings[fieldName] === "longtext"}
                <div><LongTextField bind:data={data[fieldName]}/></div>
                <!-- {:else if configuration[fieldName] === "image"}
                <div><ImageField bind:data={data[fieldName]} fileKey={fieldName}/></div> -->
                {:else if columnSettings[fieldName] === "bool"}
                <div><BooleanField bind:data={data[fieldName]}/></div>
                {:else if columnSettings[fieldName] === "sheetSelect"}
                <div><EnumField allowEmpty={(("sheetSelect@"+fieldName) in columnSettings && columnSettings[("sheetSelect@"+fieldName)].allowEmpty)} bind:data={data[fieldName]} options={config.depotInfo.sheetsFiltered.guids} aliases={config.depotInfo.sheetsFiltered.names}/></div>

                <!-- {:else if configuration[fieldName] === "multiple"}
                <div><MultipleField bind:data={data[fieldName]}/></div> -->
                {:else if columnSettings[fieldName] === "int" || columnSettings[fieldName] === "float"}
                <div><NumberField bind:data={data[fieldName]}/></div>
                
                {:else if columnSettings[fieldName] === "enum"}
                <div><EnumField allowEmpty={false} bind:data={data[fieldName]} options={columnSettings["enum@"+fieldName].options}/></div>

                <!-- column and line select assume a sheet field in the editing object -->
                {:else if columnSettings[fieldName].split("@")[0] === "lineSelect"}
                    <div><ConfigLineSelect  bind:data={data[fieldName]}
                                            bind:sheetGUID={data[columnSettings[fieldName].split("@")[1]]}
                                            config={config}
                                            /></div>
                    {#if (data["sheet"] !== "" && data["defaultValue"] !== "") && !config.depotInfo.lines[data["sheet"]].guids.includes(data[fieldName])}
                        <div>Warning: Selected value with GUID {data[fieldName]} not in selected sheet</div>
                    {/if}

                <!-- column names are inherently unique, so just bind off name instead of needing guid -->
                {:else if columnSettings[fieldName].split("@")[0] === "columnSelect"}
                    <div><ConfigColumnSelect    bind:data={data[fieldName]}
                                                sheetName={data[columnSettings[fieldName].split("@")[1]]}
                                                config={config}
                                                fieldName={fieldName}
                                                columnSettings={columnSettings}/></div>
                {/if}
            </td>
        </tr>
        {/if}
    {/each}
    </table>
    <br>
    {#if config.operation === "new"}
    <button on:click={createBlob} disabled={!validName}>Create</button>
    {:else if config.operation === "edit"}
    <button on:click={saveBlob} disabled={!validName}>Save</button>
    <button on:click={deleteBlob}>Delete</button>
    {/if}
    <button on:click={closeEditor}>Cancel</button>
    {#if debug}
    <pre>{JSON.stringify({data},null,2)}</pre>
    <pre>{JSON.stringify({config},null,2)}</pre>
    {/if}
{/if}