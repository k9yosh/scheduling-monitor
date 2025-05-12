<template>
  <div class="job-dashboard">
    <h2>Job Execution Dashboard</h2>
    <div v-if="loading">Loading initial data...</div>
    <div v-if="error" class="error">{{ error }}</div>

    <div v-if="!loading && jobExecutions.length > 0" class="job-grid">
      <div v-for="job in jobExecutions" :key="job.executionId" :class="['job-card', statusClass(job.status), { 'job-card-finishing': job.isFinishing }]"
      >
        <div class="card-header">
          <span class="job-name">{{ job.jobName }}</span>
          <span class="job-status">{{ job.status }}</span>
        </div>
        <div class="card-body">
          <p><strong>Execution ID:</strong> {{ job.executionId }}</p>
          <p><strong>Custom Name:</strong> {{ job.jobParameters?.customJobName || 'N/A' }}</p>
          <p><strong>Start Time:</strong> {{ formatDateTime(job.startTime) }}</p>
          <p><strong>End Time:</strong> {{ formatDateTime(job.endTime) }}</p>
          <p><strong>Exit Code:</strong> {{ job.exitCode || 'N/A' }}</p>
        </div>
      </div>
    </div>
    <div v-else-if="!loading && jobExecutions.length === 0" class="no-jobs">
      <svg xmlns="http://www.w3.org/2000/svg" width="64" height="64" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1" stroke-linecap="round" stroke-linejoin="round" class="feather feather-inbox">
          <polyline points="22 12 16 12 14 15 10 15 8 12 2 12"></polyline>
          <path d="M5.45 5.11L2 12v6a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2v-6l-3.45-6.89A2 2 0 0 0 16.76 4H7.24a2 2 0 0 0-1.79 1.11z"></path>
      </svg>
      <p>No job executions found.</p>
      <span>Check back later or ensure the backend job scheduler is active.</span>
    </div>

    <!-- Separator -->
    <hr class="section-separator" v-if="!loadingHistory || jobHistory.length > 0"/> 

    <!-- History Section -->
    <div class="history-section">
      <h2 v-if="!loadingHistory && jobHistory.length > 0">Recent Job History</h2>
      <div v-if="loadingHistory">Loading job history...</div>
      <div v-if="errorHistory" class="error">{{ errorHistory }}</div>
      
      <table v-if="!loadingHistory && jobHistory.length > 0" class="history-table">
        <thead>
          <tr>
            <th>Execution ID</th>
            <th>Job Name</th>
            <th>Status</th>
            <th>Start Time</th>
            <th>End Time</th>
            <th>Exit Code</th>
            <!-- Maybe add duration later -->
          </tr>
        </thead>
        <tbody>
          <tr v-for="job in jobHistory" :key="'hist-' + job.executionId" :class="statusClass(job.status)">
            <td>{{ job.executionId }}</td>
            <td>{{ job.jobName }}</td>
            <td>{{ job.status }}</td>
            <td>{{ formatDateTime(job.startTime) }}</td>
            <td>{{ formatDateTime(job.endTime) }}</td>
            <td>{{ job.exitCode }}</td>
          </tr>
        </tbody>
      </table>
       <div v-else-if="!loadingHistory && jobHistory.length === 0 && !errorHistory" class="no-jobs history">
           <p>No recent job history found.</p>
       </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted, onUnmounted, nextTick } from 'vue';

// Define the structure based on BatchJobExecutionInfoDTO.java
interface BatchJobExecutionInfo {
  executionId: number;
  jobInstanceId: number;
  jobName: string;
  status: string; // e.g., STARTING, STARTED, COMPLETED, FAILED, STOPPING, STOPPED, UNKNOWN
  startTime: string | null; // ISO 8601 format expected from backend
  endTime: string | null;   // ISO 8601 format expected from backend
  exitCode: string | null;
  exitDescription: string | null;
  jobParameters: Record<string, any> | null;
  isFinishing?: boolean; // Flag for removal animation
}

const TERMINAL_STATUSES = ['COMPLETED', 'FAILED', 'STOPPED'];
const FINISH_ANIMATION_DURATION = 500; // ms, should match CSS animation
const FINISH_VISIBILITY_DELAY = 5000; // ms, time to show terminal status before animating

export default defineComponent({
  name: 'JobDashboard',
  setup() {
    const jobExecutions = ref<BatchJobExecutionInfo[]>([]); // Active jobs (for cards)
    const jobHistory = ref<BatchJobExecutionInfo[]>([]); // Historical jobs (for table)
    const loading = ref<boolean>(true); // Loading state for active jobs
    const loadingHistory = ref<boolean>(true); // Loading state for history
    const error = ref<string | null>(null); // Error for active jobs
    const errorHistory = ref<string | null>(null); // Error for history
    const eventSource = ref<EventSource | null>(null);
    
    // Map to store pending removal timeout IDs { executionId: { visibilityTimeoutId, removalTimeoutId } }
    const pendingRemovals = ref(new Map<number, { visibilityTimeoutId: number | null, removalTimeoutId: number | null }>());

    const apiBaseUrl = 'http://localhost:8080/api/jobs';

    // --- Utility to clear timeouts for a specific job --- 
    const clearPendingRemovalTimeouts = (executionId: number) => {
      if (pendingRemovals.value.has(executionId)) {
        const timeouts = pendingRemovals.value.get(executionId)!;
        if (timeouts.visibilityTimeoutId) {
          clearTimeout(timeouts.visibilityTimeoutId);
        }
        if (timeouts.removalTimeoutId) {
          clearTimeout(timeouts.removalTimeoutId);
        }
        pendingRemovals.value.delete(executionId);
        console.log(`Cleared pending removal timeouts for ${executionId}`);
      }
    };

    const fetchSnapshot = async () => {
      loading.value = true;
      error.value = null;
      try {
        const response = await fetch(`${apiBaseUrl}/dashboard-snapshot`);
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        const data: BatchJobExecutionInfo[] = await response.json();
        // Snapshot now only contains non-terminal jobs
        jobExecutions.value = data.filter(job => !TERMINAL_STATUSES.includes(job.status))
                                  .sort((a, b) => a.executionId - b.executionId);
      } catch (err: any) {
        console.error("Error fetching snapshot:", err);
        error.value = `Failed to load initial snapshot: ${err.message}. Is the backend running?`;
      } finally {
        loading.value = false;
      }
    };

    const fetchHistory = async () => {
      loadingHistory.value = true;
      errorHistory.value = null;
      try {
        const response = await fetch(`${apiBaseUrl}/recent`);
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        const data: BatchJobExecutionInfo[] = await response.json();
        // Sort history by end time descending (most recent first), nulls last
        jobHistory.value = data.sort((a, b) => {
           const timeA = a.endTime ? new Date(a.endTime).getTime() : 0;
           const timeB = b.endTime ? new Date(b.endTime).getTime() : 0;
           // If times are the same or both null, sort by execution ID desc
           if (timeB === timeA) {
               return b.executionId - a.executionId;
           }
           return timeB - timeA; // Most recent end time first
        });
      } catch (err: any) {
        console.error("Error fetching history:", err);
        errorHistory.value = `Failed to load job history: ${err.message}.`;
      } finally {
        loadingHistory.value = false;
      }
    };

    const connectEventSource = () => {
      if (eventSource.value) {
        eventSource.value.close();
      }

      console.log("Connecting to event stream...");
      eventSource.value = new EventSource(`${apiBaseUrl}/stream`);

      eventSource.value.onmessage = (event) => {
        try {
          const jobUpdate: BatchJobExecutionInfo = JSON.parse(event.data);
          console.log("Received job update:", jobUpdate);
          error.value = null; 
          errorHistory.value = null; 

          const isTerminal = TERMINAL_STATUSES.includes(jobUpdate.status);
          const executionId = jobUpdate.executionId;
          const activeIndex = jobExecutions.value.findIndex(job => job.executionId === executionId);

          // Clear any previous pending removal timeouts for this job
          clearPendingRemovalTimeouts(executionId);

          if (!isTerminal) { 
            // --- Handle Active/Starting Job --- 
            const updatedJob = { ...jobUpdate, isFinishing: false };
            if (activeIndex !== -1) {
              jobExecutions.value[activeIndex] = { ...jobExecutions.value[activeIndex], ...updatedJob };
            } else {
              jobExecutions.value.push(updatedJob);
            }
            // Re-sort active jobs ASCENDING
            jobExecutions.value.sort((a, b) => a.executionId - b.executionId);
            
            const historyIndex = jobHistory.value.findIndex(job => job.executionId === executionId);
            if(historyIndex !== -1) jobHistory.value.splice(historyIndex, 1);

          } else { 
            // --- Handle Terminal Job --- 
            if (activeIndex !== -1) { 
              // 1. Update data IN PLACE with terminal status, isFinishing remains FALSE
              jobExecutions.value[activeIndex] = { 
                  ...jobExecutions.value[activeIndex], 
                  ...jobUpdate, 
                  isFinishing: false // Ensure it's false here
              };
              const finalJobDataForHistory = { ...jobExecutions.value[activeIndex] };
              delete finalJobDataForHistory.isFinishing;
              
              console.log(`Job ${executionId} finished. Starting ${FINISH_VISIBILITY_DELAY}ms visibility timer.`);

              // 2. Start 5s visibility timer
              const visibilityTimeoutId = setTimeout(() => {
                  // --- AFTER 5 seconds --- 
                  const currentActiveIndex = jobExecutions.value.findIndex(job => job.executionId === executionId);
                  if (currentActiveIndex !== -1) {
                      console.log(`Visibility timer elapsed for ${executionId}. Starting removal animation.`);
                      // 3. Set isFinishing to TRUE to trigger animation
                      jobExecutions.value[currentActiveIndex].isFinishing = true;
                      
                      // 4. Start animation duration timer for ACTUAL removal
                      const removalTimeoutId = setTimeout(() => {
                          console.log(`Animation timer elapsed for ${executionId}. Removing from active list.`);
                          // --- AFTER ANIMATION (0.5s) ---
                          jobExecutions.value = jobExecutions.value.filter(job => job.executionId !== executionId);
                          
                          // Add/Update in History
                          const historyIndex = jobHistory.value.findIndex(job => job.executionId === executionId);
                          if (historyIndex !== -1) {
                              jobHistory.value[historyIndex] = finalJobDataForHistory;
                          } else {
                              jobHistory.value.push(finalJobDataForHistory);
                          }
                          jobHistory.value.sort((a, b) => {
                              const timeA = a.endTime ? new Date(a.endTime).getTime() : 0;
                              const timeB = b.endTime ? new Date(b.endTime).getTime() : 0;
                              if (timeB === timeA) { return b.executionId - a.executionId; }
                              return timeB - timeA; 
                          });
                          pendingRemovals.value.delete(executionId);
                      }, FINISH_ANIMATION_DURATION); // Wait for animation

                      // Update map with removal timeout ID
                       if (pendingRemovals.value.has(executionId)) {
                           pendingRemovals.value.get(executionId)!.removalTimeoutId = removalTimeoutId as unknown as number;
                       }
                  } else {
                       console.log(`Job ${executionId} not found in active list when visibility timer elapsed.`);
                       pendingRemovals.value.delete(executionId); // Clean up if job disappeared
                  }
              }, FINISH_VISIBILITY_DELAY); // Wait 5 seconds before starting animation

              // Store the visibility timeout ID
              pendingRemovals.value.set(executionId, { visibilityTimeoutId: visibilityTimeoutId as unknown as number, removalTimeoutId: null });

            } else {
                // Terminal update for a job not currently in the active list
                const historyIndex = jobHistory.value.findIndex(job => job.executionId === executionId);
                const finalJobData = { ...jobUpdate, isFinishing: undefined }; 
                if (historyIndex !== -1) jobHistory.value[historyIndex] = finalJobData;
                else jobHistory.value.push(finalJobData);
                jobHistory.value.sort((a, b) => {
                     const timeA = a.endTime ? new Date(a.endTime).getTime() : 0;
                     const timeB = b.endTime ? new Date(b.endTime).getTime() : 0;
                     if (timeB === timeA) { return b.executionId - a.executionId; } 
                     return timeB - timeA; 
                });
            }
          }

        } catch (parseError) {
          console.error("Error processing job update:", parseError, "Data:", event.data);
          error.value = 'Error processing real-time update.';
        }
      };

      eventSource.value.onerror = (err) => {
        console.error("EventSource failed:", err);
        error.value = 'Connection to real-time updates failed. Retrying...';
        // Simple retry logic: close and reconnect after a delay
        eventSource.value?.close();
        setTimeout(connectEventSource, 5000); // Retry after 5 seconds
      };
       eventSource.value.onopen = () => {
         console.log("Event stream connected.");
         error.value = null; // Clear error on successful connection
         // Attempt to fetch initial data again on reconnect if it failed initially
         if (jobExecutions.value.length === 0 && !loading.value) fetchSnapshot();
         if (jobHistory.value.length === 0 && !loadingHistory.value) fetchHistory();
       };
    };

    const formatDateTime = (dateTimeString: string | null): string => {
      if (!dateTimeString) return 'N/A';
      try {
        return new Date(dateTimeString).toLocaleString();
      } catch (e) {
        return 'Invalid Date';
      }
    };
    
    const statusClass = (status: string): string => {
        if (!status) return '';
        return `status-${status.toLowerCase()}`;
    };

    onMounted(() => {
      fetchSnapshot();
      fetchHistory(); // Fetch initial history
      connectEventSource();
    });

    onUnmounted(() => {
      // Clear all pending timeouts when component is unmounted
      pendingRemovals.value.forEach((timeouts, executionId) => {
          clearPendingRemovalTimeouts(executionId);
      });
      pendingRemovals.value.clear();
      
      if (eventSource.value) {
        console.log("Closing event stream.");
        eventSource.value.close();
      }
    });

    return {
      jobExecutions,
      jobHistory, // Expose history data
      loading,
      loadingHistory, // Expose history loading state
      error,
      errorHistory, // Expose history error state
      formatDateTime,
      statusClass
    };
  }
});
</script>

<style scoped>
.job-dashboard {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  padding: 20px;
  background-color: #f4f7f6; /* Light background for the whole page */
  color: #333; /* Default dark text color */
}

h2 {
  color: #2c3e50;
  text-align: center;
  margin-bottom: 30px;
}

.job-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); /* Responsive grid */
  gap: 20px;
  margin-top: 15px;
}

.job-card {
  background-color: #fff; /* White background for cards */
  border-radius: 8px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  padding: 15px;
  border-left: 5px solid #ccc; /* Default border color */
  transition: transform 0.2s ease-in-out;
}

.job-card:hover {
    transform: translateY(-3px);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
  padding-bottom: 10px;
  border-bottom: 1px solid #eee;
}

.job-name {
  font-weight: bold;
  font-size: 1.1em;
  color: #34495e;
}

.job-status {
  font-weight: bold;
  padding: 3px 8px;
  border-radius: 4px;
  color: #fff;
  background-color: #95a5a6; /* Default status background */
}

.card-body p {
  margin: 5px 0;
  font-size: 0.9em;
  color: #555;
}

.card-body p strong {
    color: #333;
}

.error {
  color: #e74c3c; /* Red */
  margin-top: 10px;
  padding: 10px;
  border: 1px solid #e74c3c;
  background-color: #fdedec;
  border-radius: 4px;
}

.no-jobs {
    text-align: center;
    margin-top: 50px; /* More vertical space */
    padding: 40px 20px;
    background-color: #fff; /* Match card background */
    border-radius: 8px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05); /* Subtle shadow */
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
}

.no-jobs svg {
    color: #bdc3c7; /* Light grey icon */
    margin-bottom: 15px;
}

.no-jobs p {
    font-size: 1.2em;
    color: #555;
    margin-bottom: 5px;
}

.no-jobs span {
    font-size: 0.9em;
    color: #7f8c8d; /* Grey text */
}

/* Status-specific styling - using border-left and status badge color */
.status-completed {
  border-left-color: #2ecc71; /* Green */
}
.status-completed .job-status {
  background-color: #2ecc71;
}

@keyframes blink-effect {
  50% {
    opacity: 0.3; /* Keep the pronounced blink */
  }
}

/* Set ONLY border color for started/starting cards */
.status-started,
.status-starting {
  border-left-color: #3498db; /* Blue */
  /* REMOVE animation from here */
}

/* Set ONLY background color for started/starting badges */
.status-started .job-status,
.status-starting .job-status {
  background-color: #3498db;
}

/* Apply blinking animation ONLY to the STATUS BADGE when not finishing */
.job-card.status-started:not(.job-card-finishing) .job-status,
.job-card.status-starting:not(.job-card-finishing) .job-status {
   animation: blink-effect 1.5s linear infinite;
}

.status-failed {
  border-left-color: #e74c3c; /* Red */
}
.status-failed .job-status {
  background-color: #e74c3c;
}

.status-stopped {
    border-left-color: #f39c12; /* Orange */
}
.status-stopped .job-status {
    background-color: #f39c12;
}

.status-unknown {
    border-left-color: #bdc3c7; /* Grey */
}
.status-unknown .job-status {
    background-color: #95a5a6;
}

.section-separator {
    border: none;
    border-top: 1px solid #e0e0e0;
    margin: 40px 0;
}

.history-section h2 {
    text-align: left;
    margin-bottom: 15px;
    font-size: 1.5em;
    color: #34495e;
}

.history-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 15px;
  background-color: #fff;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  border-radius: 5px; /* Slight rounding for table */
  overflow: hidden; /* Ensures border-radius clips content */
}

.history-table th, .history-table td {
  border: 1px solid #e0e0e0;
  padding: 10px 12px;
  text-align: left;
  font-size: 0.9em;
}

.history-table th {
  background-color: #f8f9fa;
  font-weight: 600;
  color: #495057;
}

.history-table tbody tr:nth-child(even) {
  background-color: #f9f9f9;
}

/* Reuse status classes for table rows, maybe just background? */
.history-table .status-completed td:first-child {
  border-left: 3px solid #2ecc71; /* Green */
}
.history-table .status-started td:first-child,
.history-table .status-starting td:first-child {
  border-left: 3px solid #3498db; /* Blue */
}
.history-table .status-failed td:first-child {
  border-left: 3px solid #e74c3c; /* Red */
}
.history-table .status-stopped td:first-child {
    border-left: 3px solid #f39c12; /* Orange */
}
.history-table .status-unknown td:first-child {
    border-left: 3px solid #bdc3c7; /* Grey */
}

/* Override blinking for table rows if applied via class */
.history-table tr {
    animation: none !important; 
}

.no-jobs.active {
    /* Styles specific to no *active* jobs message */
}

.no-jobs.history {
    /* Styles specific to no *history* message */
    margin-top: 20px; /* Less margin than the main one */
    padding: 20px;
    box-shadow: none;
    background-color: transparent;
}

/* Animation for card removal */
@keyframes fadeOutShrink {
  from {
    opacity: 1;
    transform: scale(1);
  }
  to {
    opacity: 0;
    transform: scale(0.9);
  }
}

.job-card-finishing {
  animation: fadeOutShrink 0.5s ease-out forwards;
  /* Prevent interaction while fading */
  pointer-events: none; 
}
</style> 