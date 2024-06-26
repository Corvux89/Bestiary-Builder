<template>
	<button class="btn open-btn" @click="showFeatureModal = true" :id="type + index">Edit Feature</button>
	<Modal :show="showFeatureModal" @close="showFeatureModal = false" :full-screen="true">
		<template #header> Edit feature: {{ feat.name }} </template>
		<template #body>
			<div class="two-wide uneven">
				<div>
					<div class="editor-field__container two-wide">
						<LabelledComponent title="Feature name">
							<input type="text" id="featurename" placeholder="Enter name" v-model="feat.name" @change="hasEditedName = true" />
						</LabelledComponent>
						<LabelledComponent title="Documentation">
							<div>
								<a href="https://avrae.readthedocs.io/en/stable/automation_ref.html"> Documentation</a> <br>
								<a href="https://avrae.io/dashboard/characters" v-tooltip="'Edit automation on a character to get access to the full fledged automation builder.'"> Automation Editor</a>
							</div>
						</LabelledComponent>
					</div>


					<div class="editor-field__container">
						<LabelledComponent title="Feature description">
							<textarea height="94" id="featuredescription" placeholder="Enter description" v-model="feat.description" style="height: 93px;"/>
						</LabelledComponent>
					</div>

					<div class="editor-field__container two-wide">
						<LabelledComponent title="Save automation">
							<button class="btn confirm" @click="saveAutomation(true)" id="saveautomation">Save </button>
						</LabelledComponent>
						<LabelledComponent title="Generate automation">
							<button 
								class="btn" 
								@click="generateAutomation" 
								v-tooltip="'Generate automation from description. May be incomplete or inaccurate. Only works for basic, to hit attacks.'"
								id="generateautomation"
							>
								Generate 
								<font-awesome-icon :icon="['fas', 'circle-info']" />
							</button>
						</LabelledComponent>
					</div>

					<div class="editor-field__container two-wide" v-if="isMobile">
						<LabelledComponent title="Clear automation">
							<button class="btn danger" @click="automationString = 'null'" id="clearautomation">Clear</button>
						</LabelledComponent>
						<LabelledComponent title="Copy Automation">
							<button 
								class="btn" 
								@click="copyAutomation()" 
								id="copyautomation"
							>
								Copy 
							</button>
						</LabelledComponent>
					</div>

					<hr />
					<div class="editor-field__container">
						<LabelledComponent title="Import SRD feature">
							<v-select :options="srdFeatures" v-model="importedSrdFeature" inputId="importsrdfeature" />
							<button class="btn move-down" @click="importSrdAction">Load</button>
						</LabelledComponent>
					</div>

					<div class="editor-field__container">
						<LabelledComponent title="Import basic example">
							<v-select :options="basicExamples" v-model="importedBasicExample" inputId="importbasicexample" />
							<button class="btn move-down" @click="importExample">Load</button>
						</LabelledComponent>
					</div>

					<div v-if="errorMessage">
						<hr />
						<span class="yaml-error" v-html="errorMessage"> </span>
					</div>
					<div v-if="showDescriptionButtons && (errorMessage == null || errorMessage?.length == 0)">
						<hr />
						<p class="warning"> Feature description and last automation text node do not match. </p>
						<p> You can update the other with the buttons here.</p>
						<div class="two-wide editor-field__container">
							<LabelledComponent title="Update from automation">
								<button class="btn" @click="updateFeatureDescFromAutomationDesc"> Update </button>
							</LabelledComponent>
							<LabelledComponent title="Update from description">
								<button class="btn" @click="updateAutomationDescFromFeatureDesc"> Update </button>
							</LabelledComponent>
						</div>
					</div>
				</div>
				<div class="automation-editor">
					<VueMonacoEditor
						v-model:value="automationString"
						theme="vs-dark"
						:options="{wordWrap: 'on', theme: 'vs-dark', minimap: {enabled: false}, formatOnPaste: true, formatOnType: true, automaticLayout: true, scrollBeyondLastLine: false}"
						height="600px"
						:language="getLanguage"
						@mount="handleMount"
					/>
				</div>
			</div>
			<div class="two-wide uneven">
				<div></div>
				<div v-if="currentDocu" class="docs">
					<hr>
					<h3> Documentation: {{ currentContext }}</h3>
					<p class="small" v-html="md.render(currentDocu.desc)" />

					<div >
						<hr>
						<h4> Overview </h4>
						See full documentation <a :href="'https://avrae.readthedocs.io/en/stable/automation_ref.html#' + currentDocu.url" target="_blank">here</a>.
						<VueMonacoEditor 
							v-if="currentDocu?.ts"
							:value="'// Values denoted with an ? are optional.\n' + currentDocu.ts"
							theme="vs-dark"
							:options="{wordWrap: 'on', theme: 'vs-dark', minimap: {enabled: false}, automaticLayout: true, readOnly: true, scrollBeyondLastLine: false}"
							language="json"
							height="200px"
						/>
					</div>
					<div v-if="currentDocu?.opt">
						<hr>
						<h4> Options </h4>
						<ul>
							<li v-for="info, name in currentDocu.opt">
								<span>
									<code class="highlight">{{ name }}</code> 
									<span v-html="md.render(info)"/> 
								</span>
							</li>
						</ul>
					</div>
					<div v-if="currentDocu?.variables">
						<hr>
						<h4> Exposed Variables </h4>
						<ul>
							<li v-for="info, name in currentDocu.variables">
								<span><span class="highlight">{{ name }}</span> [<code>{{ info.type }}</code>] <span v-html="md.render(info.desc)" /> </span>
							</li>
						</ul>
					</div>
				</div>
			</div>

		</template>
	</Modal>
</template>

<script lang="ts">
import {defineComponent, shallowRef, ref, onUnmounted} from "vue";
import { VueMonacoEditor } from '@guolao/vue-monaco-editor'
import YAML from "yaml";
import {toast, handleApiResponse, type error} from "@/main";
import {type FeatureEntity} from "@/../../shared";
import LabelledComponent from "./LabelledComponent.vue";
import Modal from "./Modal.vue";
import {parseDescIntoAutomation} from "@/parser/utils";
import markdownit from "markdown-it"
import { isMobile } from "@/main";
const md = markdownit()
export default defineComponent({
	props: ["type", "index", "data"],
	data() {
		return {
			feat: this.data.features[this.type][this.index],
			automationString: "" as string,
			errorMessage: null as null | string,
			basicExamples: [] as string[],
			srdFeatures: [] as string[],
			docu: {} as Record<string, unknown>,
			importedBasicExample: null as string | null,
			importedSrdFeature: null as string | null,
			showFeatureModal: false,
			hasEditedName: false,
			currentContext: "",
			md,
			YAML,
			isMobile
		};
	},
	components: {
		LabelledComponent,
		VueMonacoEditor,
		Modal
	},
	setup() {
		const editorRef  = shallowRef()
		const handleMount = (editor: any) => (editorRef.value = editor)

		let cursorPosition = ref(0);

		let ourInterval = setInterval(() => {
			cursorPosition.value = editorRef.value?.getModel().getOffsetAt(editorRef.value?.getPosition());
		}, 1000)
		
		onUnmounted(() => {
			clearInterval(ourInterval)
		})
		return {
			handleMount,
			editorRef,
			cursorPosition
		}
	},
	mounted() {
		this.automationString = YAML.stringify(this.feat.automation) ?? YAML.stringify(null);

		//Fetch example and feature namesP
		fetch("/api/basic-examples/list").then(async (response: any) => {
			let result = await handleApiResponse<string[]>(response);
			if (result.success) this.basicExamples = result.data as string[];
			else this.basicExamples = [];
		});
		fetch("/api/srd-features/list").then(async (response: any) => {
			let result = await handleApiResponse<string[]>(response);
			if (result.success) this.srdFeatures = result.data as string[];
			else this.srdFeatures = [];
		});

		fetch("/api/automationDocumentation").then(async (response: any) => {
			let result = await handleApiResponse<Record<string, unknown>>(response);
			if (result.success) this.docu = result.data as Record<string, unknown>;
			else this.docu = {};
		});
	},
	methods: {
		validateYaml(): boolean {
			try {
				YAML.parse(this.automationString);
				this.errorMessage = null;
				return true;
			} catch (err) {
				// @ts-ignore
				this.errorMessage = err;
				return false;
			}
		},
		saveAutomation(shouldNotify: boolean = false) {
			try {
				let parsed = YAML.parse(this.automationString);
				if (parsed) {
					fetch("/api/validate/automation", {
						method: "POST",
						headers: {
							"Content-Type": "application/json"
						},
						body: JSON.stringify({
							data: parsed
						})
					})
						.then((response) => response.json())
						.then((data) => {
							if (!data.success) {
								if (shouldNotify) toast.error("YAML contains Error, did not save automation");
								this.errorMessage = data.error;
							} else {
								this.feat.automation = YAML.parse(this.automationString);
								if (shouldNotify) toast.success("Saved Automation!");
							}
						});
				} else {
					this.feat.automation = YAML.parse(this.automationString);
					if (shouldNotify) toast.success("Saved Automation!");
				}
			} catch (err) {
				if (shouldNotify) toast.error("YAML contains Error, did not save automation");
			}
		},
		async importExample() {
			if (!this.importedBasicExample) return;
			//Fetch
			let example = null as FeatureEntity | null;
			await fetch("/api/basic-example/" + encodeURIComponent(this.importedBasicExample)).then(async (response: any) => {
				let result = await handleApiResponse<FeatureEntity>(response);
				if (result.success) example = result.data as FeatureEntity;
				else {
					toast.error("Error: " + (result.data as error).error);
					example = null;
				}
			});
			if (!example) return;
			//Add info to feat
			if (this.feat.name == "New Feature" || !this.hasEditedName) this.feat.name = example.name;
			this.feat.description = example.description ?? "";
			this.automationString = YAML.stringify(example.automation);
			this.saveAutomation(false);
			setTimeout(() => {
				let els = document.querySelectorAll(".language-yaml") as NodeListOf<HTMLElement>;
				for (let e in els) {
					if (els[e].dataset?.highlighted == "yes") els[e].dataset.highlighted = "";
				}
			}, 100);

			toast.success("Successfully loaded: " + example.name);
		},
		async importSrdAction() {
			if (!this.importedSrdFeature) return;
			//Fetch
			let feature = null as FeatureEntity | null;
			await fetch("/api/srd-feature/" + encodeURIComponent(this.importedSrdFeature)).then(async (response: any) => {
				let result = await handleApiResponse<FeatureEntity>(response);
				if (result.success) feature = result.data as FeatureEntity;
				else {
					toast.error("Error: " + (result.data as error).error);
					feature = null;
				}
			});

			if (!feature) return;
			//Add info to feat, making sure to clean it all up.
			if (this.feat.name == "New Feature" || !this.hasEditedName) {
				if (feature.automation != null) this.feat.name = feature.name.split('-').slice(1).join('-').trim();
				else this.feat.name = feature.name
			} 
			this.feat.description = feature.description.replaceAll("$NAME$", this.data.description.name) ?? "";

			if (Array.isArray(feature.automation)) {
				for (let feat of feature.automation) {
					feat['name'] = feat['name'].split('-').slice(1).join('-').trim();
				}
			} else if (feature.automation != null) {
				// @ts-ignore
				feature.automation['name'] = feature.automation['name'].split('-').slice(1).join('-').trim();
			} 
			
			this.automationString = YAML.stringify(feature.automation);
			this.saveAutomation(false);
			setTimeout(() => {
				let els = document.querySelectorAll(".language-yaml") as NodeListOf<HTMLElement>;
				for (let e in els) {
					if (els[e].dataset?.highlighted == "yes") els[e].dataset.highlighted = "";
				}
			}, 100);

			toast.success("Successfully loaded: " + feature.name);
		},
		generateAutomation() {
			const result = parseDescIntoAutomation(this.feat.description, this.feat.name, 0)[0];
			if (result) {
				try {
					this.automationString = YAML.stringify(result);
					this.saveAutomation(false);
				} catch {
					toast.error("Something went when generating automation!");
				}
			}
		},
		getAutomationDescription() : string | false {
			let auto;
			try {
				auto = YAML.parse(this.automationString)
			} catch {
				return false;
			}
			if (Array.isArray(auto)) return false;
			if (!this.feat.automation || !auto || auto.automation.length == 0) return false
			for (let field of auto.automation.reverse()) {
				if (field["type"] == "text") {
					return field["text"]
				}
			}
			return ""
		},
		updateFeatureDescFromAutomationDesc() : void {
			let auto;
			try {
				auto = YAML.parse(this.automationString)
			} catch {
				return;
			}			if (Array.isArray(auto)) return;
			for (let field of auto.automation.reverse()) {
				if (field["type"] == "text") {
					this.feat.description = field["text"]
					return;
				}
			}
		},
		updateAutomationDescFromFeatureDesc() : void { 
			let auto;
			try {
				auto = YAML.parse(this.automationString)
			} catch {
				return;
			}			if (Array.isArray(auto)) return;
			for (let field of auto.automation.reverse()) {
				if (field["type"] == "text") {
					field["text"] = this.feat.description
					auto.automation.reverse()
					this.automationString = YAML.stringify(auto)
					return;
				}
			}
		},
		copyAutomation()  {
			navigator.clipboard.writeText(this.automationString);
			toast.success("Copied automation to clipboard!")
		},
		getContext() {
			const textToTraverse = this.automationString;
			let buffer = ""
			let type = ""
			let startingPosition = this.cursorPosition;

			// if the user has their cursor on the word type, it would begin the buffer in that word and would not be able to detect it. 
			// Therefore before we start we check if the immediate vicinity of our cursor includes type:. if it does, start 6 characters later so we can properly extract the type.
			// clamp slices to string min and max, to be sure it doesn't go out of range
			const closeVicinity = textToTraverse.slice(Math.max(startingPosition - 6, 0 ), Math.min(startingPosition + 6, textToTraverse.length))
			if (closeVicinity.includes("type:")) startingPosition += 6

			// from our position in the string we go backwards until we find the first type: string.
			// Then we get our first word after that type:
			// Works both with yaml or json string
			// Assumes that type: is always at the beginning of a node, and that a user does not have type: in a text field
			for (let i = startingPosition; i--; i < textToTraverse.length) {
				const char = textToTraverse.charAt(i)
				buffer = char + buffer
				if (buffer.startsWith("type:")) {
					type = textToTraverse.slice(i).match(/type['"]?:\s*['"]?(\w+)['"]?/)?.[1] || ""
					break;
				} 
			}

			this.currentContext = type
		},
	},
	computed: {
		showDescriptionButtons() : boolean {
			const desc = this.feat.description
			const autoDesc = this.getAutomationDescription()
			if (Array.isArray(this.feat.automation) || !desc || !autoDesc) return false
			if (desc != autoDesc) return true
			return false
		},
		getLanguage() : string {
			if (this.automationString.startsWith('{') || this.automationString.startsWith('[')) return "json"
			return "yaml"
		},
		currentDocu() : any {
			return (this.docu[this.currentContext] as Record<string, unknown>)
		},
	},
	watch: {
		automationString() {
			this.validateYaml();

			if (this.feat.name == "New Feature" || this.feat.description == "") {
				try {
					let parsedAutomation = YAML.parse(this.automationString);
					if (Array.isArray(parsedAutomation)) {
						if (parsedAutomation.length > 0) parsedAutomation = parsedAutomation[0];
						else return;
					}
					///console.log(parsedAutomation);

					if (parsedAutomation.name && this.feat.name == "New Feature") this.feat.name = parsedAutomation.name.replace(" (1H)", "").replace(" (2H)", "");

					if (parsedAutomation.automation && this.feat.description == "") {
						for (let type in parsedAutomation.automation) {
							if (parsedAutomation.automation[type]["type"] == "text") {
								///console.log(parsedAutomation.automation[type]["text"]);
								this.feat.description = parsedAutomation.automation[type]["text"];
								break;
							}
						}
					}
				} catch {}
			}

			// I hate this.
			let els = document.querySelectorAll(".language-yaml") as NodeListOf<HTMLElement>;
			for (let e in els) {
				if (els[e].dataset?.highlighted == "yes") els[e].dataset.highlighted = "";
			}
		},
		"feat.description"() {
			// I hate this.
			let els = document.querySelectorAll(".language-markdown") as NodeListOf<HTMLElement>;
			for (let e in els) {
				if (els[e].dataset?.highlighted == "yes") els[e].dataset.highlighted = "";
			}
		},
		cursorPosition() {
			this.getContext()
		}
	}
});
</script>

<style scoped lang="less">
.automation-editor {
	max-width: 1fr;
}
.open-btn {
	width: 100%;
}

a {
	color: orangered;
}
.two-wide {
	display: grid;
	gap: 2rem;
	grid-template-columns: 1fr 1fr;

	&.uneven {
		grid-template-columns: 1fr 2fr;
		max-width: 100%;
	}
}

.three-wide {
	display: grid;
	gap: 2rem;
	grid-template-columns: 1fr 1fr 1fr;
}
@media screen and (max-width: 1080px) {
	.three-wide {
		grid-template-columns: 1fr 1fr;
	}
}

@media screen and (max-width: 950px) {
	.three-wide,
	.two-wide {
		gap: 1rem;
		grid-template-columns: 1fr;
	}
}

@media screen and (max-width: 1660px) {
	.two-wide.uneven {
		gap: 1rem;
		grid-template-columns: 1fr 1fr;
	}
}

@media screen and (max-width: 1200px) {
	.two-wide.uneven {
		gap: 1rem;
		grid-template-columns: 1fr;
	}
}



.move-down {
	margin-top: 0.4rem;
}



.editor-field__container {
	display: grid;
	gap: 1rem 2rem;
	margin-bottom: 1rem;

	.flow-vertically {
		display: flex;
		flex-direction: column;
		gap: 0.3rem;

		.button-container p {
			display: flex;
			justify-content: space-between;
		}
	}

	.editor-field__title .text {
		font-weight: bold;
		text-decoration: underline;
	}


	textarea {
		min-height: 46px;
		height: 46px;
	}
}

.highlight {
	color: orangered;
	border-left: 3px solid orangered;
	padding: 3px;
}
</style>

<style lang="less">
.yaml-error {
	color: var(--color-destructive);
	display: flex;
	flex-direction: column;
	font-weight: bold
}

.validation-error-header {
	margin: 0 0 0.5em 0;
}

.validation-error-list {
	margin: 0 0 0.25em 0;
}

.validation-error-item {
	margin: 0;
}

.docs a {
	color: orangered;
}
</style>
