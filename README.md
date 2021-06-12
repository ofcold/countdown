# countdown
Typescript Countdown component for Vue 3.


## Basic usage

```vue
<template>
	<countdown :time="2 * 24 * 60 * 60 * 1000" v-slot="{ days, hours, minutes, seconds }">
		Time Remaining：{{ days }} days, {{ hours }} hours, {{ minutes }} minutes, {{ seconds }} seconds.
	</countdown>
</template>
```

## Custom interval

```vue
<template>
	<countdown :time="time" :interval="100" v-slot="{ days, hours, minutes, seconds, milliseconds }">
		New Year Countdown：{{ days }} days, {{ hours }} hours, {{ minutes }} minutes, {{ seconds }}.{{ Math.floor(milliseconds / 100) }} seconds.
	</countdown>
</template>

<script>
import {defineComponent, ref} from 'vue'
export default defineComponent({
	setup() {
		const now = new Date();
		const newYear = ref(new Date(now.getFullYear() + 1, 0, 1));

		return {
			time: newYear - now,
		};
	}
})
</script>

```

## Countdown on demand

```vue

<template>
	<button type="button" class="btn btn-primary" :disabled="counting" @click="startCountdown">
		<countdown v-if="counting" :time="60000" @end="onCountdownEnd" v-slot="{ totalSeconds }">Fetch again {{ totalSeconds }} seconds later</countdown>
		<span v-else>Fetch Verification Code</span>
	</button>
</template>

<script>
import {ref, defineComponent} from 'vue'
export default defineComponent({
	setup() {
		const counting = ref(false)
		function startCountdown() {
			counting = true
		}
		function onCountdownEnd() {
			counting = false
		}

		return {
			counting,
			startCountdown,
			onCountdownEnd
		}
	}
})
</script>

```

