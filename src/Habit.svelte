<script>
	import {debugLog, isValidCSSColor} from './utils'

	import {onDestroy} from 'svelte'
	import {parseYaml, TFile} from 'obsidian'
	import {getDateAsString, getDayOfTheWeek} from './utils'
	import {addDays, differenceInCalendarDays, parseISO} from 'date-fns'

	export let app
	export let name
	export let path
	export let dates
	export let debug
	export let pluginName
	export let userSettings
	export let globalSettings

	let entries = []
	let frontmatter = {}
	let habitName = name
	let customStyles = ''

	// Reactive color resolution - updates whenever frontmatter, userSettings, or globalSettings change
	$: {
		const resolvedColor =
			frontmatter.color || userSettings.color || globalSettings.defaultColor
		if (resolvedColor && isValidCSSColor(resolvedColor)) {
			customStyles = `--habit-bg-ticked: ${resolvedColor}`
		} else {
			customStyles = ''
		}
	}
	const computeStreakIntervals = function (entries, maxGap) {
		if (!entries.length) return []
		const gap = Number(maxGap)
		const sorted = [...entries].sort()
		const intervals = []
		let iStart = 0

		for (let i = 1; i < sorted.length; i++) {
			const gapDays =
				differenceInCalendarDays(parseISO(sorted[i]), parseISO(sorted[i - 1])) -
				1
			if (gapDays > gap) {
				const lastTick = sorted[i - 1]
				intervals.push({
					start: sorted[iStart],
					end: getDateAsString(addDays(parseISO(lastTick), gap)),
					count: i - iStart,
				})
				iStart = i
			}
		}

		// Last interval — extend by maxGap days (same as broken intervals)
		const lastEntry = sorted[sorted.length - 1]
		intervals.push({
			start: sorted[iStart],
			end: getDateAsString(addDays(parseISO(lastEntry), gap)),
			count: sorted.length - iStart,
		})
		return intervals
	}

	$: entriesInRange = (() => {
		const maxGap = frontmatter.maxGap ?? globalSettings.maxGap

		if (!maxGap) {
			return dates.reduce((acc, date) => {
				const streak = findStreak(date)
				acc[date] = {
					ticked: entries.includes(date),
					streak,
					inStreak: streak > 0,
					isStreakStart: streak === 1,
					isStreakEnd: null, // determined in getClasses for the no-max_gap path
				}
				return acc
			}, {})
		}

		const intervals = computeStreakIntervals(entries, maxGap)
		return dates.reduce((acc, date) => {
			const ticked = entries.includes(date)
			const interval = intervals.find(
				(iv) => date >= iv.start && date <= iv.end,
			)
			if (interval) {
				acc[date] = {
					ticked,
					streak: date === interval.end ? interval.count : 0,
					inStreak: true,
					isStreakStart: date === interval.start,
					isStreakEnd: date === interval.end,
				}
			} else {
				acc[date] = {
					ticked,
					streak: 0,
					inStreak: false,
					isStreakStart: false,
					isStreakEnd: false,
				}
			}
			return acc
		}, {})
	})()

	let savingChanges = false

	$: getClasses = function (date) {
		let classes = [
			'habit-tracker__cell',
			`habit-tracker__cell--${getDayOfTheWeek(date)}`,
			'habit-tick',
		]

		const {ticked, streak, inStreak, isStreakStart, isStreakEnd} =
			entriesInRange[date]

		if (ticked) {
			classes.push('habit-tick--ticked')
		}

		// Only add streak classes if streaks are enabled
		const showStreaksEnabled =
			userSettings.showStreaks !== undefined
				? userSettings.showStreaks
				: globalSettings.showStreaks
		if (showStreaksEnabled) {
			if (inStreak) {
				classes.push('habit-tick--streak')
			}
			if (inStreak && !ticked) {
				classes.push('habit-tick--streak-gap')
			}
			if (isStreakStart) {
				classes.push('habit-tick--streak-start')
			}

			const maxGap = frontmatter.maxGap ?? globalSettings.maxGap
			if (maxGap) {
				if (isStreakEnd) {
					classes.push('habit-tick--streak-end')
				}
			} else {
				let isNextDayTicked = false
				const nextDate = getDateAsString(addDays(parseISO(date), 1))
				if (date === dates.at(-1)) {
					// last in the dates in range
					isNextDayTicked = entries.includes(nextDate)
				} else {
					isNextDayTicked = entriesInRange[nextDate].ticked
				}
				if (ticked && !isNextDayTicked) {
					classes.push('habit-tick--streak-end')
				}
			}
		}

		return classes.join(' ')
	}

	const findStreak = function (date) {
		let currentDate = parseISO(date)
		let streak = 0

		while (entries.includes(getDateAsString(currentDate))) {
			streak++
			currentDate.setDate(currentDate.getDate() - 1)
		}

		return streak
	}

	const init = async function () {
		debugLog(`Loading habit ${habitName}`, debug, undefined, pluginName)

		const getFrontmatter = async function (path) {
			const file = this.app.vault.getAbstractFileByPath(path)

			if (!file || !(file instanceof TFile)) {
				debugLog(
					`No file found for path: ${path}`,
					debug,
					undefined,
					pluginName,
				)
				return {}
			}

			try {
				return await this.app.vault.read(file).then((result) => {
					const frontmatter = result.split('---')[1]

					if (!frontmatter) {
						return {entries: []}
					}
					const fmParsed = parseYaml(frontmatter)
				if (fmParsed['entries'] == undefined) {
						fmParsed['entries'] = []
					}

					return fmParsed
				})
			} catch (error) {
				debugLog(
					`Error in habit ${habitName}: error.message`,
					debug,
					undefined,
					pluginName,
				)
				return {}
			}
		}

		frontmatter = await getFrontmatter(path)
		debugLog(`Frontmatter for ${path} ↴`, debug)
		debugLog(frontmatter, debug)
		entries = frontmatter.entries
		entries = entries.sort()
		habitName = frontmatter.title || habitName

		debugLog(`Habit "${habitName}": Found ${entries.length} entries`, debug)
		debugLog(entries, debug, undefined, pluginName)
	}

	const toggleHabit = function (date) {
		const file = this.app.vault.getAbstractFileByPath(path)
		if (!file || !(file instanceof TFile)) {
			new Notice(`${pluginName}: file missing while trying to toggle habit`)
			return
		}

		let newEntries = [...entries]
		if (entriesInRange[date].ticked) {
			newEntries = newEntries.filter((e) => e !== date)
		} else {
			newEntries.push(date)
		}
		entries = newEntries.sort()

		savingChanges = true

		this.app.fileManager.processFrontMatter(file, (frontmatter) => {
			frontmatter['entries'] = entries
		})
	}

	init()

	const modifyRef = app.vault.on('modify', (file) => {
		if (file.path === path) {
			if (!savingChanges) {
				console.log('oh shit, i was modified')
				init()
			}
			savingChanges = false
		}
	})

	onDestroy(() => {
		app.vault.offref(modifyRef)
	})
</script>

<!-- <div bind:this={rootElement}> -->
<div
	class="habit-tracker__row"
	style={customStyles}
>
	<div class="habit-tracker__cell--name habit-tracker__cell">
		<a
			href={path}
			aria-label={path}
			class="internal-link">{habitName}</a
		>
	</div>
	{#if Object.keys(entriesInRange).length}
		{#each dates as date}
			<!-- svelte-ignore a11y-no-static-element-interactions -->
			<!-- svelte-ignore a11y-click-events-have-key-events -->
			<div
				class={getClasses(date)}
				ticked={entriesInRange[date].ticked}
				streak={entriesInRange[date].streak}
				on:click={() => toggleHabit(date)}
			></div>
		{/each}
	{/if}
</div>
