<template>
  <div class="control-panel">
    <h2>Control Panel</h2>
    <div class="job-controls">
      <div v-for="job in availableJobs" :key="job.jobName" class="control-card">
        <h3>{{ job.displayName }}</h3>
        <div class="duration-selector">
          <label>Duration:</label>
          <div class="options">
            <label v-for="duration in durationOptions" :key="duration">
              <input 
                type="radio" 
                :name="'duration-' + job.jobName" 
                :value="duration" 
                v-model="job.selectedDuration"
              />
              {{ duration }}s
            </label>
          </div>
        </div>
        <button @click="launchJob(job.jobName, job.selectedDuration)" :disabled="job.isLoading">
          <svg v-if="!job.isLoading" xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="currentColor" class="play-icon"><path d="M8 5v14l11-7z"></path></svg>
          <span v-else class="loading-spinner"></span>
          Launch
        </button>
        <p v-if="job.launchStatus" :class="['launch-status', job.launchStatus.type]">
            {{ job.launchStatus.message }}
        </p>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref } from 'vue';

interface JobControl {
  jobName: string;
  displayName: string;
  selectedDuration: number;
  isLoading: boolean;
  launchStatus: { type: 'success' | 'error'; message: string } | null;
}

export default defineComponent({
  name: 'ControlPanel',
  setup() {
    const apiBaseUrl = 'http://localhost:8080/api/jobs';
    const durationOptions = ref([10, 20, 30, 60]);

    const availableJobs = ref<JobControl[]>([
      { jobName: 'simulatedJob', displayName: 'Simulated Job 1 - COMPLETE', selectedDuration: 10, isLoading: false, launchStatus: null },
      { jobName: 'simulatedJob2', displayName: 'Simulated Job 2 - COMPLETE', selectedDuration: 10, isLoading: false, launchStatus: null },
      { jobName: 'simulatedJob3', displayName: 'Simulated Job 3 - COMPLETE', selectedDuration: 10, isLoading: false, launchStatus: null },
      { jobName: 'simulatedJob4', displayName: 'Simulated Job 4 - FAILURE', selectedDuration: 10, isLoading: false, launchStatus: null },
      { jobName: 'simulatedJob5', displayName: 'Simulated Job 5 - STOPPED', selectedDuration: 10, isLoading: false, launchStatus: null },
    ]);

    const launchJob = async (jobName: string, duration: number) => {
      const job = availableJobs.value.find(j => j.jobName === jobName);
      if (!job) return;

      job.isLoading = true;
      job.launchStatus = null; // Clear previous status

      console.log(`Launching ${jobName} with duration ${duration}s`);

      try {
        const response = await fetch(`${apiBaseUrl}/launch/${jobName}`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({ durationInSeconds: duration }),
        });

        if (response.status === 202) { // Accepted
          const result = await response.json(); // Backend returns BatchJobExecutionInfoDTO
          console.log(`Job ${jobName} launched successfully, execution ID: ${result.executionId}`);
          job.launchStatus = { type: 'success', message: `Launched (ID: ${result.executionId})` };
        } else {
          const errorText = await response.text();
          throw new Error(`Launch failed: ${response.status} - ${errorText}`);
        }
      } catch (error: any) {
        console.error(`Error launching job ${jobName}:`, error);
         job.launchStatus = { type: 'error', message: `Error: ${error.message}` };
      } finally {
        job.isLoading = false;
        // Optionally clear status message after a delay
        setTimeout(() => {
           if(job.launchStatus) job.launchStatus = null; 
        }, 5000); 
      }
    };

    return {
      availableJobs,
      durationOptions,
      launchJob,
    };
  },
});
</script>

<style scoped>
.control-panel {
  padding: 20px;
  background-color: #e9ecef; /* Lighter grey background */
  border-right: 1px solid #ced4da;
  min-width: 280px; /* Minimum width - adjust if needed */
  max-width: 300px; /* Reduced maximum width */
  overflow-y: auto; /* Allow scrolling if needed */
}

h2 {
  color: #495057;
  text-align: center;
  margin-bottom: 20px;
  font-size: 1.4em;
}

.job-controls {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.control-card {
  background-color: #fff;
  border-radius: 6px;
  padding: 15px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.control-card h3 {
  margin-top: 0;
  margin-bottom: 15px;
  font-size: 1.1em;
  color: #343a40;
}

.duration-selector {
  margin-bottom: 15px;
}

.duration-selector label {
    display: block;
    margin-bottom: 5px;
    font-weight: bold;
    font-size: 0.9em;
    color: #6c757d;
}

.duration-selector .options {
    display: flex;
    gap: 10px;
    flex-wrap: wrap; /* Allow wrapping if needed */
}

.duration-selector .options label {
    font-weight: normal;
    display: flex;
    align-items: center;
    cursor: pointer;
    font-size: 0.9em;
}

.duration-selector input[type="radio"] {
    margin-right: 5px;
}

button {
  display: inline-flex; /* Align icon and text */
  align-items: center;
  justify-content: center;
  padding: 8px 15px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.9em;
  transition: background-color 0.2s ease;
  min-width: 90px; /* Ensure button has some width */
}

button:hover {
  background-color: #0056b3;
}

button:disabled {
  background-color: #6c757d;
  cursor: not-allowed;
}

.play-icon {
  margin-right: 5px;
  width: 1em; /* Scale with font size */
  height: 1em;
}

.loading-spinner {
  display: inline-block;
  width: 1em;
  height: 1em;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  border-top-color: #fff;
  animation: spin 1s ease-in-out infinite;
  margin-right: 5px;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.launch-status {
    font-size: 0.85em;
    margin-top: 10px;
    padding: 5px 8px;
    border-radius: 4px;
}
.launch-status.success {
    color: #155724;
    background-color: #d4edda;
    border: 1px solid #c3e6cb;
}
.launch-status.error {
    color: #721c24;
    background-color: #f8d7da;
    border: 1px solid #f5c6cb;
}

</style> 