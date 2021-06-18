<template>
	<slot v-bind="{days, hours, minutes, seconds, milliseconds, totalDays, totalHours, totalMinutes, totalSeconds, totalMilliseconds: data.totalMilliseconds}"/>
</template>
<script lang="ts">
import { defineComponent, reactive, computed, watchEffect, onMounted, onBeforeUnmount} from 'vue';

const MILLISECONDS_SECOND = 1000;
const MILLISECONDS_MINUTE = 60 * MILLISECONDS_SECOND;
const MILLISECONDS_HOUR = 60 * MILLISECONDS_MINUTE;
const MILLISECONDS_DAY = 24 * MILLISECONDS_HOUR;

interface CountdownInterface {
	/**
	 * It is counting down.
	 *
	 * @type {boolean}
	 */
	counting: boolean,

	/**
	 * The absolute end time.
	 *
	 * @type {number}
	 */
	endTime: number,

	/**
	 * The remaining milliseconds.
	 *
	 * @type {number}
	 */
	totalMilliseconds: number,

	/**
	 * The request ID of the requestAnimationFrame.
	 *
	 * @type {number}
	 */
	requestId: number,
}

export default defineComponent({
	name: 'countdown',
	props: {
		// Starts the countdown automatically when initialized.
		autoStart: {
			type: Boolean,
			default: true,
		},

		// Emits the countdown events.
		emitEvents: {
			type: Boolean,
			default: true,
		},

		// The interval time (in milliseconds) of the countdown progress.
		interval: {
			type: Number,
			default: 1000,
			validator: (value: number) => value >= 0,
		},

		// Generate the current time of a specific time zone.
		now: {
			type: Function,
			default: () => Date.now(),
		},

		// The time (in milliseconds) to count down from.
		time: {
			type: Number,
			default: 0,
			validator: (value: number) => value >= 0,
		},
	},

	setup({autoStart, emitEvents, interval, now, time }, {emit}) {

		const data = reactive<CountdownInterface>({
			counting: false,
			endTime: 0,
			totalMilliseconds: 0,
			requestId: 0,
		});

		/**
		 * The computed values.
		 *
		 * @return {[number]}
		 */
		const days = computed((): number => Math.floor(data.totalMilliseconds / MILLISECONDS_DAY));
		const hours = computed((): number => Math.floor((data.totalMilliseconds % MILLISECONDS_DAY) / MILLISECONDS_HOUR));
		const minutes = computed((): number => Math.floor((data.totalMilliseconds % MILLISECONDS_HOUR) / MILLISECONDS_MINUTE));
		const seconds = computed((): number => Math.floor((data.totalMilliseconds % MILLISECONDS_MINUTE) / MILLISECONDS_SECOND));
		const milliseconds = computed((): number => Math.floor(data.totalMilliseconds % MILLISECONDS_SECOND));
		const totalDays = computed((): number => days);
		const totalHours = computed((): number => Math.floor(data.totalMilliseconds / MILLISECONDS_HOUR));
		const totalMinutes = computed((): number => Math.floor(data.totalMilliseconds / MILLISECONDS_MINUTE));
		const totalSeconds = computed((): number => Math.floor(data.totalMilliseconds / MILLISECONDS_SECOND));

		watchEffect(() => {
			data.totalMilliseconds = time;
			data.endTime = now() + time;

			if (autoStart) {
				start();
			}
		});

		onMounted((): void => {
			document.addEventListener('visibilitychange', handleVisibilityChange);
		});

		onBeforeUnmount((): void => {
			document.removeEventListener('visibilitychange', handleVisibilityChange);
			pause();
		});

		/**
		 * Starts to countdown.
		 *
		 * @public
		 *
		 * @emits Countdown#start
		 */
		function start() {
			if (data.counting) {
				return;
			}

			data.counting = true;

			if (emitEvents) {
				// Countdown start event.
				emit('start');
			}

			if (document.visibilityState === 'visible') {
				continuance();
			}
		}

		/**
		 * Continues the countdown.
		 *
		 * @private
		 */
		function continuance() {
			if (! data.counting) {
				return;
			}

			const delay = Math.min(data.totalMilliseconds, interval);

			if (delay > 0) {
				let init: number;
				let prev: number;
				const step = (_now: number) => {
					if (!init) {
						init = _now;
					}

					if (!prev) {
						prev = _now;
					}

					const range = _now - init;

					if (
						range >= delay

						// Avoid losing time about one second per minute (_now - prev ≈ 16ms) (#43)
						|| range + ((_now - prev) / 2) >= delay
					) {
						progress();
					} else {
						data.requestId = requestAnimationFrame(step);
					}

					prev = _now;
				};

				data.requestId = requestAnimationFrame(step);
			} else {
				end();
			}
		}

		/**
		 * Pauses the countdown.
		 *
		 * @private
		 */
		function pause () {
			cancelAnimationFrame(data.requestId)
		}

		/**
		 * Progresses to countdown.
		 *
		 * @private
		 *
		 * @emits Countdown#progress
		 */
		function progress () {
			if (! data.counting) {
				return;
			}

			data.totalMilliseconds -= interval;

			if (emitEvents && data.totalMilliseconds > 0) {
				// Countdown progress event.
				emit('progress', {
					days: days,
					hours: hours,
					minutes: minutes,
					seconds: seconds,
					milliseconds: milliseconds,
					totalDays: totalDays,
					totalHours: totalHours,
					totalMinutes: totalMinutes,
					totalSeconds: totalSeconds,
					totalMilliseconds: data.totalMilliseconds,
				});
			}

			continuance();
		}

		/**
		 * Aborts the countdown.
		 *
		 * @public
		 *
		 * @emits Countdown#abort
		 */
		function abort() {
			if (! data.counting) {
				return;
			}

			pause();
			data.counting = false;

			if (emitEvents) {
				// Countdown abort event.
				emit('abort');
			}
		}

		/**
		 * Ends the countdown.
		 *
		 * @public
		 *
		 * @emits Countdown#end
		 */
		function end() {
			if (! data.counting) {
				return;
			}

			pause();

			data.totalMilliseconds = 0;
			data.counting = false;

			if (emitEvents) {
				// Countdown end event.
				emit('end');
			}
		}

		/**
		 * Updates the count.
		 *
		 * @private
		 */
		function update() {
			if (data.counting) {
				data.totalMilliseconds = Math.max(0, data.endTime - now());
			}
		}

		/**
		 * visibility change event handler.
		 *
		 * @private
		 */
		function handleVisibilityChange() {
			switch (document.visibilityState) {
				case 'visible':
					update();
					continuance();
					break;

				case 'hidden':
					pause();
					break;
			}
		}

		return {
			data,
			// Computed
			days, hours, minutes, seconds, milliseconds, totalDays, totalHours, totalMinutes, totalSeconds,

			// Methods
			start, continuance, pause, progress, abort, end, update, handleVisibilityChange,
		};
	}
})
</script>
