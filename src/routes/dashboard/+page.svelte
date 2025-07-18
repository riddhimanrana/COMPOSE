<script lang="js">
	import { useChat } from "ai/svelte";
	import { supabase } from "$lib/supabaseClient";
	import { get } from "svelte/store";
	import ProblemList from "$lib/components/ProblemList.svelte";
	import ProgressBar from "$lib/components/ProgressBar.svelte";
	import Button from "$lib/components/Button.svelte";
	import { Checkbox, TextArea } from "carbon-components-svelte";
	import { onMount } from "svelte";
	import toast from "svelte-french-toast";
	import { handleError } from "$lib/handleError";
	import { fetchSettings } from "$lib/supabase/settings";
	import {
		getImages,
		getProblemCounts,
		getThisUser,
		getThisUserRole,
		getProblems,
		getProblemFeedback,
	} from "$lib/supabase";
	import { List, Schematics } from "carbon-icons-svelte";
	import { Dropdown, MultiSelect } from "carbon-components-svelte";

	const datasetPrompt = `
		The database you have access to is a view called full_problems. English descriptions of the database columns with each column type in parenthesis are given below:
			answer_latex (string | null): The answer to the problem written in LaTeX;
			archived (boolean | null): Whether the problem has been archived;
			author_id (string | null): Supabase ID of the user who wrote the problem;
			comment_latex (string | null): Comments given by the author of the problem written in LaTeX;
			created_at (string | null): Timestamp problem was created at;
			difficulty (number | null): Difficulty rating of the problem;
			edited_at (string | null): Timestamp the problem was last edited at;
			front_id (string | null): An identifier for each problem given by the first 3 letters of the author's full name with the id number;
			full_name (string | null): Name of the author of the problem;
			id (number | null): Unique ID number of the problem;
			nickname (string | null): Nickname for each problem;
			problem_latex (string | null): The problem written in LaTeX;
			problem_tests (string | null): Comma-separated list of all tests that the problem appears on written as a single string;
			solution_latex (string | null): The solution to the problem written in LaTeX;
			sub_topics (string | null): Comma-separated list of topics which appear in the problem – sub-topics are more granular than topics and tend to cover tactics or themes present in the problem and solution;
			topics (string | null): Comma-separated list of the overall topics that appear in the problem – all topics are chosen from Algebra, Calculus, Combinatorics, Number Theory, Geometry;
			topics_short (string | null): The same as the topics field but the names are shortened to Alg, Calc, Combo, NT, Geo for Algebra, Calculus, Combinatorics, Number Theory, Geometry respectively;
			unresolved_count (number | null): The number of unresolved pieces of feedback the problem has;
	`;

	const promptParts = [
		"COMPOSE - the Collaborative Online Math Problem Organization and Sharing Environment - is a storage platform for contest math problems.",
		"Math contest organizers must query the COMPOSE database when creating math competitions.",
		"You are CASSIE - the COMPOSE AI Support System and Information Expert.",
		"Your job is to answer user's questions regarding the COMPOSE database to the best of your knowledge.",
		"Each entry in the database corresponds to one math problem.",
		datasetPrompt,
		"Database queries should fill in the [TODO] in the following supabase-js function template: ```javascript await supabase.from('full_problems').select('*').[TODO]```",
		//"This user's ID is " + user.id,
		"If your message includes a database query, do not include any additional text.",
	];

	const { input, handleSubmit, messages, isLoading } = useChat({
		initialMessages: [
			{
				role: "system",
				content: promptParts.join(" "),
			},
		],
		onFinish: processLastMessage,
	});

	let problems = [];
	let filteredProblems = [];

	let time_filtered_problems = [];
	let problemCounts = [];
	let width = 0;
	let loaded = false;
	let user;
	let userRole;

	let openModal = false;
	let values = ["Problems", "Answers", "Solutions", "Comments"];
	let group = values.slice(0, 1);

	let scheme = {};

	// Test filtering variables
	let allTests = [];
	let selectedTests = [];
	let sortBy = "difficulty";
	let sortDirection = "asc";

	onMount(async () => {
		// Fetch settings from the database
		scheme = await fetchSettings();
	});

	(async () => {
		try {
			user = await getThisUser();
			userRole = await getThisUserRole();
			problems = await getProblems({ customEq: { author_id: user.id } });
			sortProblems();
			extractTests();
			filteredProblems = [...problems];
			applyFiltersAndSort();
			console.log(scheme.progress.after);
			time_filtered_problems = await getProblems({
				after: new Date(scheme.progress.after),
				before: new Date(scheme.progress.before),
				customEq: { author_id: user.id },
			});
			console.log(time_filtered_problems.length);

			const topicsCount = problems.reduce((count, { topics }) => {
				let individualTopics;
				if (topics) {
					individualTopics = topics.split(", ").map((topic) => topic.trim());
				} else {
					individualTopics = ["Uncategorized"];
				}

				individualTopics.forEach((topic) => {
					count[topic] = (count[topic] || 0) + 1;
				});

				return count;
			}, {});
			console.log(topicsCount);
			const problemCountsData = await getProblemCounts();
			console.log(problemCountsData);
			const sortedKeys = Object.keys(topicsCount)
				.filter((key) => key !== "Uncategorized")
				.sort();
			if ("Uncategorized" in topicsCount) {
				sortedKeys.push("Uncategorized");
			}
			problemCounts = sortedKeys.reduce((sortedObj, key) => {
				sortedObj[key] = topicsCount[key];
				return sortedObj;
			}, {});

			loaded = true;
		} catch (error) {
			handleError(error);
			toast.error(error.message);
		}
	})();

	function submitWrapper(e) {
		loaded = false;
		handleSubmit(e);
	}

	function processLastMessage() {
		console.log(messages);
		const allMessages = get(messages);
		console.log(messages);
		const curMessage = allMessages[allMessages.length - 1];
		console.log(curMessage);
		if (
			curMessage.role == "assistant" &&
			curMessage.content.includes("await supabase")
		) {
			console.log(curMessage.content);
			const regex = /```javascript(.*?)```/s;
			const match = curMessage.content.match(regex);
			console.log(match);
			let codeBlock = curMessage.content;
			if (match) {
				codeBlock = match[1].trim();
			}
			const asyncFunction = new Function(
				"supabase",
				`
                    return (async () => {
                        const { data } = ${codeBlock}
                        return data;
                    })();
                    `
			);
			try {
				const result = asyncFunction(supabase)
					.then((result) => {
						console.log(result);
						problems = result;
						extractTests();
						applyFiltersAndSort();
						console.log("Async code execution completed.");
					})
					.catch((error) => {
						console.error("Async code execution error:", error);
					});
				console.log("Code executed successfully. Result:", result);
			} catch (error) {
				console.error("Error executing code:", error);
			}
		} else {
			console.log("Not a function");
		}

		loaded = true;
	}

	// Reactive statements to update filters when dependencies change
	$: if (problems.length > 0 && (selectedTests || sortBy || sortDirection)) {
		applyFiltersAndSort();
	}

	function sortProblems() {
		console.log("SORTING");
		// TODO: updated stages
		const statusOrder = [
			"Archived",
			"Published",
			"Draft",
			"Idea",
			"Endorsed",
			"On Test",
		];
		const stageOrder = [
			"Needs Review",
			"Awaiting Feedback",
			"Awaiting Endorsement",
			"Awaiting Testsolve",
			"Testsolve Received",
			"Complete",
		];
		problems = problems.sort((a, b) => {
			const statusComparison =
				statusOrder.indexOf(b.status) - statusOrder.indexOf(a.status);
			console.log(statusComparison);
			if (statusComparison !== 0) {
				return statusComparison; // Sort by status first
			}
		});
	}

	function extractTests() {
		const testCounts = {};
		problems.forEach((problem) => {
			if (problem.problem_tests) {
				const tests = problem.problem_tests
					.split(", ")
					.map((test) => test.trim());
				tests.forEach((test) => {
					testCounts[test] = (testCounts[test] || 0) + 1;
				});
			}
		});

		const sortedTests = Object.keys(testCounts).sort();
		allTests = sortedTests.map((test) => ({
			id: test,
			text: `${test} (${testCounts[test]})`,
			count: testCounts[test],
		}));
	}

	function applyFiltersAndSort() {
		if (!problems || problems.length === 0) {
			filteredProblems = [];
			return;
		}

		// Filter by test
		let filtered = [...problems];
		if (selectedTests.length > 0) {
			filtered = problems.filter((problem) => {
				if (!problem.problem_tests) return false;
				const tests = problem.problem_tests
					.split(", ")
					.map((test) => test.trim());
				return selectedTests.some((selectedTest) =>
					tests.includes(selectedTest)
				);
			});
		}

		// Sort by selected criteria
		filtered = filtered.sort((a, b) => {
			let comparison = 0;

			if (sortBy === "difficulty") {
				const diffA = a.difficulty || 0;
				const diffB = b.difficulty || 0;
				comparison = diffA - diffB;
			} else if (sortBy === "topics") {
				const topicsA = a.topics || "";
				const topicsB = b.topics || "";
				comparison = topicsA.localeCompare(topicsB);
			}

			return sortDirection === "asc" ? comparison : -comparison;
		});

		filteredProblems = filtered;
	}

	async function getBucketPaths(path) {
		try {
			const data = await getImages(path);
			let ans = [];
			for (let i = 0; i < data.length; i++) {
				if (data[i].id != null) {
					if (path === "") {
						ans.push(data[i].name);
					} else {
						ans.push(path + "/" + data[i].name);
					}
				} else {
					let x;
					if (path === "") {
						x = await getBucketPaths(data[i].name);
					} else {
						x = await getBucketPaths(path + "/" + data[i].name);
					}
					for (let j = 0; j < x.length; j++) {
						ans.push(x[j]);
					}
				}
			}
			return ans;
		} catch (error) {
			handleError(error);
			toast.error(error.message);
		}
	}

	async function openProblemsPDF() {
		try {
			let l =
				"\\title{All Problems}\\date{Mustang Math}\\begin{document}\\maketitle";

			for (const problem of problems) {
				l += "\\section*{Problem " + problem.front_id + "}";
				if (group.includes("Problems")) {
					l +=
						"\\textbf{Problem:} " +
						problem.problem_latex +
						"\\newline\\newline";
				}

				if (group.includes("Answers") && problem.answer_latex != "") {
					l +=
						"\\textbf{Answer:} " + problem.answer_latex + "\\newline\\newline";
				}

				if (group.includes("Solutions") && problem.solution_latex != "") {
					l +=
						"\\textbf{Solution:} " +
						problem.solution_latex.replace("\\ans{", "\\boxed{") +
						"\\newline\\newline";
				}

				if (group.includes("Comments") && problem.comment_latex != "") {
					l +=
						"\\textbf{Comment:} " +
						problem.comment_latex.replace("\\ans{", "\\boxed{") +
						"\\newline\\newline";
				}
			}
			l += "\\end{document}";

			let images = await getBucketPaths("");

			const resp = await fetch(
				// make env variable before pushing
				import.meta.env.VITE_PDF_GENERATOR_URL,
				{
					method: "POST",
					headers: {
						"Content-Type": "application/json",
						mode: "no-cors",
					},
					body: JSON.stringify({
						latex: l,
						images,
					}),
				}
			);
			const blob = await resp.blob();
			const newBlob = new Blob([blob]);
			const blobUrl = window.URL.createObjectURL(newBlob);
			const link = document.createElement("a");
			link.href = blobUrl;
			link.setAttribute("download", "problems.pdf");
			document.body.appendChild(link);
			link.click();
			link.parentNode.removeChild(link);

			// clean up Url
			window.URL.revokeObjectURL(blobUrl);
		} catch (error) {
			handleError(error);
			toast.error(error.message);
		}
	}
</script>

<svelte:window bind:outerWidth={width} />

<br />
<h1>Your Dashboard</h1>
{#if !loaded}
	<p>Loading problems...</p>
{/if}

<div style="margin-top: 10px;">
	<Button title="Create a new problem" href="/problems/new" />
</div>
{#if userRole >= 30}
	<div style="margin-top: 10px;">
		<Button title="Problem Inventory" href="/problems" />
	</div>
{/if}
<br />
<div class="flex">
	<div class="stats">
		<h4><u>Your Stats</u></h4>
		{#if loaded && false}
			/**
			<ProgressBar
				value={time_filtered_problems.length}
				max={scheme.progress.goal}
				helperText={time_filtered_problems.length +
					"/" +
					scheme.progress.goal +
					" problems written"}
				labelText={"Tournament Progress"}
			/>
			*/
		{/if}
		<p>
			<strong>Number of Problems: {problems.length}</strong>
		</p>
		{#each Object.entries(problemCounts) as [cat, count]}
			<p>
				<!-- prettier-ignore -->
				<strong>{cat} Problems:</strong>
				{count}
			</p>
		{/each}
	</div>
</div>
<br />
<ul visibility: hidden>
	{#each $messages as message}
		<li>{message.role}: {message.content}</li>
	{/each}
</ul>
<br />

{#if loaded}
	<div class="filter-container">
		<div class="filter-header">
			<h3>Filter & Sort Problems</h3>
			{#if selectedTests.length > 0 || sortBy !== "difficulty" || sortDirection !== "asc"}
				<Button
					title="Clear Filters"
					on:click={() => {
						selectedTests = [];
						sortBy = "difficulty";
						sortDirection = "asc";
					}}
					kind="ghost"
					size="small"
				/>
			{/if}
		</div>

		<div class="filter-controls">
			<div class="filter-group">
				<MultiSelect
					titleText="Filter by Tests"
					label="Select tests..."
					bind:selectedIds={selectedTests}
					items={allTests}
					sortItem={() => {}}
				/>
			</div>

			<div class="filter-group">
				<Dropdown
					titleText="Sort by"
					bind:selectedId={sortBy}
					items={[
						{ id: "difficulty", text: "Difficulty" },
						{ id: "topics", text: "Topics" },
					]}
				/>
			</div>

			<div class="filter-group">
				<Dropdown
					titleText="Direction"
					bind:selectedId={sortDirection}
					items={[
						{ id: "asc", text: "Ascending" },
						{ id: "desc", text: "Descending" },
					]}
				/>
			</div>

			<div class="results-count">
				<p>
					Showing <strong>{filteredProblems.length}</strong> of
					<strong>{problems.length}</strong> problems
				</p>
				{#if selectedTests.length > 0}
					<div class="filter-tags">
						{#each selectedTests as test}
							<span class="filter-tag">
								<i class="fa-solid fa-filter" />
								{test}
							</span>
						{/each}
					</div>
				{/if}
			</div>
		</div>
	</div>
{/if}

<br />
<div style="width:80%; margin: auto;margin-bottom: 20px;">
	<ProblemList
		problems={loaded ? filteredProblems : problems}
		showList={JSON.parse(localStorage.getItem("problem-list.show-list"))}
		sortKey={"feedback_status"}
		sortDirection={"ascending"}
	/>
</div>

{#if openModal}
	<div
		class="flex"
		style="background-color: rgba(0,0,0,0.5); position: absolute; top: 0; bottom: 0;right:0;left: 0;z-index: 100;"
	>
		<div
			style="width: 300px; height: max-content; z-index: 101;background-color: white;padding: 10px;position: relative;"
		>
			<div style="position: absolute; top: 5px; right: 8px;">
				<button
					on:click={() => {
						openModal = !openModal;
					}}
					style="font-size: 12px;cursor:pointer;outline: none;border: none;background: none;"
					><i class="fa-solid fa-x" /></button
				>
			</div>

			<p><strong>PDF Options</strong></p>

			{#each values as value}
				<Checkbox bind:group labelText={value} {value} />
			{/each}

			<br />
			<button on:click={openProblemsPDF}>Download Problems</button>
			<br /><br />
		</div>
	</div>
{/if}

<style>
	.stats {
		background-color: white;
		border: 1px solid var(--primary);
		width: 80%;
		margin: 10px;
		text-align: left;
		padding: 10px;
	}

	.filter-container {
		width: 80%;
		margin: 20px auto;
		background-color: white;
		border: 1px solid var(--primary);
		border-radius: 8px;
		padding: 20px;
	}

	.filter-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 20px;
		border-bottom: 1px solid #e0e0e0;
		padding-bottom: 10px;
	}

	.filter-header h3 {
		margin: 0;
		color: var(--primary);
	}

	.filter-controls {
		display: flex;
		gap: 20px;
		align-items: end;
		flex-wrap: wrap;
	}

	.filter-group {
		min-width: 180px;
		flex: 1;
	}

	.results-count {
		margin-left: auto;
		text-align: right;
		display: flex;
		flex-direction: column;
		gap: 5px;
	}

	.results-count p {
		margin: 0;
		color: var(--primary);
	}

	.filter-tags {
		display: flex;
		flex-wrap: wrap;
		gap: 5px;
	}

	.filter-tag {
		background-color: var(--primary);
		color: white;
		padding: 4px 8px;
		border-radius: 4px;
		font-size: 12px;
		white-space: nowrap;
	}

	.filter-tag i {
		margin-right: 4px;
	}

	@media (max-width: 768px) {
		.filter-controls {
			flex-direction: column;
			align-items: stretch;
		}

		.results-count {
			margin-left: 0;
			text-align: left;
			margin-top: 15px;
		}

		.filter-group {
			min-width: 100%;
		}
	}
</style>
