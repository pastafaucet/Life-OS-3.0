# Timer Feature Implementation Specification
**For Claude Code / Visual Studio Code Development**

## Overview
Add time tracking functionality to the existing legal task entry system. This adds a start/stop timer button and actual time badge to each task.

## Implementation Instructions
Modify the existing `legal-tasks.html` file with the EXACT changes specified below. Do NOT modify anything else.

## Step 1: Add Timer Styles
Add these styles to the existing `<style>` section, right before the closing `</style>` tag:

```css
/* Timer Button */
.timer-button {
    width: 20px;
    height: 20px;
    margin-right: 8px;
    background: none;
    border: none;
    color: #6b7280;
    cursor: pointer;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    font-size: 14px;
    transition: all 0.2s ease;
    padding: 0;
    flex-shrink: 0;
}

.timer-button:hover {
    transform: scale(1.1);
    color: #9ca3af;
}

.timer-button.running {
    color: #3b82f6;
}

/* Timer Badge */
.timer-badge {
    background-color: rgba(168, 85, 247, 0.1);
    color: #a855f7;
}

.timer-badge.empty {
    background-color: rgba(168, 85, 247, 0.05);
    color: #6b7280;
    border: 1px dashed #374151;
}

.timer-badge.empty:hover {
    color: #a855f7;
    border-color: #a855f7;
}

/* Timer Modal */
.timer-modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.8);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
}

.timer-modal {
    background: #1a1a1a;
    border: 1px solid #2a2a2a;
    border-radius: 12px;
    padding: 24px;
    width: 400px;
    max-width: 90vw;
}

.timer-modal h3 {
    margin: 0 0 20px 0;
    color: #ffffff;
    font-size: 18px;
    font-weight: 600;
}

.timer-modal-row {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-bottom: 16px;
}

.timer-modal-row label {
    flex: 0 0 80px;
    color: #9ca3af;
    font-size: 14px;
}

.timer-modal-row input {
    flex: 1;
    background: #0a0a0a;
    border: 1px solid #2a2a2a;
    border-radius: 6px;
    padding: 8px 12px;
    color: #ffffff;
    font-size: 14px;
    outline: none;
}

.timer-modal-row input:focus {
    border-color: #3b82f6;
}

.quick-buttons {
    display: flex;
    gap: 8px;
    margin: 20px 0;
    flex-wrap: wrap;
}

.quick-button {
    padding: 6px 12px;
    background: #0a0a0a;
    border: 1px solid #2a2a2a;
    border-radius: 6px;
    color: #9ca3af;
    cursor: pointer;
    font-size: 13px;
    transition: all 0.2s ease;
}

.quick-button:hover {
    border-color: #3b82f6;
    color: #3b82f6;
}

.timer-total {
    text-align: center;
    margin: 20px 0;
    font-size: 20px;
    color: #a855f7;
    font-weight: 600;
}

.timer-modal-actions {
    display: flex;
    gap: 12px;
    justify-content: flex-end;
    margin-top: 24px;
}

.timer-modal-actions button {
    padding: 8px 16px;
    border-radius: 6px;
    border: none;
    font-size: 14px;
    cursor: pointer;
    transition: all 0.2s ease;
}

.timer-modal-actions .save-btn {
    background: #3b82f6;
    color: #ffffff;
}

.timer-modal-actions .save-btn:hover {
    background: #2563eb;
}

.timer-modal-actions .cancel-btn {
    background: #1f2937;
    color: #9ca3af;
    border: 1px solid #374151;
}

.timer-modal-actions .cancel-btn:hover {
    background: #374151;
}

.timer-modal-actions .clear-btn {
    background: #ef4444;
    color: #ffffff;
}

.timer-modal-actions .clear-btn:hover {
    background: #dc2626;
}

/* Adjust task title to accommodate timer button */
.task-header {
    display: flex;
    align-items: center;
    margin-bottom: 8px;
}
```

## Step 2: Update Task Object Structure
In the `TaskParser.parse()` method, update the task object initialization to include timer fields:

```javascript
const task = {
    originalInput: input,
    type: 'task', // default
    title: '',
    client: '',
    case: '',
    project: '',
    priority: 'strategic',
    doDate: '',
    hours: '',
    // Add these timer fields:
    timerStart: null,
    timerElapsed: 0,
    timerRunning: false,
    actualTime: 0
};
```

## Step 3: Add Timer Manager Class
Add this new class right after the `TaskStorage` class:

```javascript
// Timer management
class TimerManager {
    static timers = new Map();
    static intervals = new Map();

    static startTimer(taskId) {
        // Stop any other running timers
        this.stopAllTimers();

        const now = Date.now();
        this.timers.set(taskId, now);

        // Update task
        const data = TaskStorage.get();
        const task = data.tasks.find(t => t.id === taskId);
        if (task) {
            task.timerStart = now;
            task.timerRunning = true;
            TaskStorage.save(data);
        }

        // Start interval for UI updates
        const interval = setInterval(() => {
            const elapsed = this.getElapsed(taskId);
            this.updateTimerDisplay(taskId, elapsed);
        }, 1000);

        this.intervals.set(taskId, interval);
    }

    static stopTimer(taskId) {
        const interval = this.intervals.get(taskId);
        if (interval) {
            clearInterval(interval);
            this.intervals.delete(taskId);
        }

        const data = TaskStorage.get();
        const task = data.tasks.find(t => t.id === taskId);
        if (task && task.timerRunning) {
            const elapsed = this.getElapsed(taskId);
            task.timerElapsed = (task.timerElapsed || 0) + elapsed;
            task.timerRunning = false;
            task.timerStart = null;
            task.actualTime = task.timerElapsed;
            TaskStorage.save(data);
        }

        this.timers.delete(taskId);
    }

    static stopAllTimers() {
        const data = TaskStorage.get();
        data.tasks.forEach(task => {
            if (task.timerRunning) {
                this.stopTimer(task.id);
            }
        });
    }

    static toggleTimer(taskId) {
        const data = TaskStorage.get();
        const task = data.tasks.find(t => t.id === taskId);
        if (!task) return;

        if (task.timerRunning) {
            this.stopTimer(taskId);
        } else {
            this.startTimer(taskId);
        }
    }

    static getElapsed(taskId) {
        const startTime = this.timers.get(taskId);
        if (!startTime) return 0;
        return Math.floor((Date.now() - startTime) / 1000);
    }

    static getTotalElapsed(taskId) {
        const data = TaskStorage.get();
        const task = data.tasks.find(t => t.id === taskId);
        if (!task) return 0;

        const previousElapsed = task.timerElapsed || 0;
        const currentElapsed = task.timerRunning ? this.getElapsed(taskId) : 0;
        return previousElapsed + currentElapsed;
    }

    static formatTime(seconds) {
        if (seconds < 3600) {
            // Under 1 hour: show as M:SS
            const mins = Math.floor(seconds / 60);
            const secs = seconds % 60;
            return `${mins}:${secs.toString().padStart(2, '0')}`;
        } else {
            // Over 1 hour: show as Xh Ym
            const hours = Math.floor(seconds / 3600);
            const mins = Math.floor((seconds % 3600) / 60);
            return `${hours}h ${mins}m`;
        }
    }

    static updateTimerDisplay(taskId, elapsed) {
        const badge = document.querySelector(`[data-timer-id="${taskId}"]`);
        if (badge) {
            const data = TaskStorage.get();
            const task = data.tasks.find(t => t.id === taskId);
            if (task) {
                const total = (task.timerElapsed || 0) + elapsed;
                badge.textContent = `⏲ ${this.formatTime(total)}`;
            }
        }
    }

    static restoreRunningTimers() {
        const data = TaskStorage.get();
        data.tasks.forEach(task => {
            if (task.timerRunning && task.timerStart) {
                // Calculate elapsed time since timer started
                const elapsed = Math.floor((Date.now() - task.timerStart) / 1000);
                this.timers.set(task.id, task.timerStart);
                
                // Start interval for UI updates
                const interval = setInterval(() => {
                    const currentElapsed = this.getElapsed(task.id);
                    this.updateTimerDisplay(task.id, currentElapsed);
                }, 1000);
                
                this.intervals.set(task.id, interval);
            }
        });
    }
}
```

## Step 4: Replace the createTaskElement Method
Replace the ENTIRE `createTaskElement` method in the `UIManager` class with this:

```javascript
createTaskElement(task) {
    const div = document.createElement('div');
    div.className = 'task-item';
    
    // Task header with timer button and title
    const header = document.createElement('div');
    header.className = 'task-header';
    
    // Timer button
    const timerBtn = document.createElement('button');
    timerBtn.className = `timer-button ${task.timerRunning ? 'running' : ''}`;
    timerBtn.innerHTML = task.timerRunning ? '⏸' : '▶';
    timerBtn.onclick = (e) => {
        e.stopPropagation();
        TimerManager.toggleTimer(task.id);
        this.render();
    };
    header.appendChild(timerBtn);
    
    // Title
    const title = document.createElement('div');
    title.className = 'task-title';
    title.textContent = task.title || 'Untitled task';
    title.style.flex = '1';
    title.onclick = () => this.editTitle(task.id, title);
    header.appendChild(title);
    
    div.appendChild(header);

    // Badges container
    const badges = document.createElement('div');
    badges.className = 'task-badges';

    // Always show all badges in order
    
    // 1. Type badge (task/event/deadline)
    badges.appendChild(this.createBadge(
        (task.type || 'task'),
        `type-badge type-${task.type || 'task'}`,
        () => this.cycleType(task.id)
    ));
    
    // 2. Priority badge (always has a value)
    badges.appendChild(this.createBadge(
        `!${task.priority}`,
        `priority-badge priority-${task.priority}`,
        () => this.cyclePriority(task.id)
    ));

    // 3. Client badge
    badges.appendChild(this.createBadge(
        task.client ? `@${task.client}` : `@ --`,
        task.client ? 'client-badge' : 'client-badge empty',
        (badge) => this.editBadge(task.id, 'client', badge)
    ));

    // 4. Case badge
    badges.appendChild(this.createBadge(
        task.case ? `#${task.case}` : `# --`,
        task.case ? 'case-badge' : 'case-badge empty',
        (badge) => this.editBadge(task.id, 'case', badge)
    ));

    // 5. Project badge
    badges.appendChild(this.createBadge(
        task.project ? `^${task.project}` : `^ --`,
        task.project ? 'project-badge' : 'project-badge empty',
        (badge) => this.editBadge(task.id, 'project', badge)
    ));

    // 6. Date badge
    badges.appendChild(this.createBadge(
        task.doDate ? `📅 ${task.doDate}` : `📅 --`,
        task.doDate ? 'date-badge' : 'date-badge empty',
        (badge) => this.editBadge(task.id, 'doDate', badge)
    ));

    // 7. Hours badge
    badges.appendChild(this.createBadge(
        task.hours ? `⏱ ${task.hours}` : `⏱ --`,
        task.hours ? 'hours-badge' : 'hours-badge empty',
        (badge) => this.editBadge(task.id, 'hours', badge)
    ));

    // 8. Timer badge (actual time)
    const totalElapsed = TimerManager.getTotalElapsed(task.id);
    const timerBadge = this.createBadge(
        totalElapsed > 0 ? `⏲ ${TimerManager.formatTime(totalElapsed)}` : `⏲ --`,
        totalElapsed > 0 ? 'timer-badge' : 'timer-badge empty',
        (badge) => this.showTimerModal(task.id)
    );
    timerBadge.setAttribute('data-timer-id', task.id);
    badges.appendChild(timerBadge);

    div.appendChild(badges);

    // Delete button
    const deleteBtn = document.createElement('button');
    deleteBtn.className = 'delete-btn';
    deleteBtn.innerHTML = '✕';
    deleteBtn.onclick = () => {
        if (task.timerRunning) {
            TimerManager.stopTimer(task.id);
        }
        TaskStorage.deleteTask(task.id);
        this.render();
    };
    div.appendChild(deleteBtn);

    return div;
}
```

## Step 5: Add Timer Modal Methods
Add these methods to the `UIManager` class, right after the `createInlineEdit` method:

```javascript
showTimerModal(taskId) {
    const data = TaskStorage.get();
    const task = data.tasks.find(t => t.id === taskId);
    if (!task) return;

    // Create modal overlay
    const overlay = document.createElement('div');
    overlay.className = 'timer-modal-overlay';
    
    // Create modal
    const modal = document.createElement('div');
    modal.className = 'timer-modal';
    
    modal.innerHTML = `
        <h3>Adjust Timer for Task</h3>
        <div class="timer-modal-row">
            <label>Started:</label>
            <input type="time" id="timer-start-time" />
            <input type="date" id="timer-start-date" />
        </div>
        <div class="timer-modal-row">
            <label>Ended:</label>
            <input type="time" id="timer-end-time" />
            <input type="date" id="timer-end-date" />
        </div>
        <div class="quick-buttons">
            <button class="quick-button" data-minutes="-0">Just now</button>
            <button class="quick-button" data-minutes="-15">15m ago</button>
            <button class="quick-button" data-minutes="-30">30m ago</button>
            <button class="quick-button" data-minutes="-60">1h ago</button>
        </div>
        <div class="timer-total">Total time: <span id="timer-total">0:00</span></div>
        <div class="timer-modal-actions">
            <button class="clear-btn">Clear</button>
            <button class="cancel-btn">Cancel</button>
            <button class="save-btn">Save</button>
        </div>
    `;
    
    overlay.appendChild(modal);
    document.body.appendChild(overlay);
    
    // Set initial values
    const now = new Date();
    const today = now.toISOString().split('T')[0];
    const currentTime = now.toTimeString().slice(0, 5);
    
    document.getElementById('timer-start-date').value = today;
    document.getElementById('timer-end-date').value = today;
    
    if (task.timerRunning) {
        // Timer is running
        const startTime = new Date(task.timerStart);
        document.getElementById('timer-start-time').value = startTime.toTimeString().slice(0, 5);
        document.getElementById('timer-start-date').value = startTime.toISOString().split('T')[0];
        document.getElementById('timer-end-time').value = currentTime;
    } else if (task.timerElapsed > 0) {
        // Timer has been used
        const endTime = now;
        const startTime = new Date(now.getTime() - (task.timerElapsed * 1000));
        document.getElementById('timer-start-time').value = startTime.toTimeString().slice(0, 5);
        document.getElementById('timer-start-date').value = startTime.toISOString().split('T')[0];
        document.getElementById('timer-end-time').value = endTime.toTimeString().slice(0, 5);
        document.getElementById('timer-end-date').value = endTime.toISOString().split('T')[0];
    } else {
        // Timer never used
        document.getElementById('timer-start-time').value = currentTime;
        document.getElementById('timer-end-time').value = currentTime;
    }
    
    // Update total time display
    const updateTotal = () => {
        const startDate = document.getElementById('timer-start-date').value;
        const startTime = document.getElementById('timer-start-time').value;
        const endDate = document.getElementById('timer-end-date').value;
        const endTime = document.getElementById('timer-end-time').value;
        
        if (startDate && startTime && endDate && endTime) {
            const start = new Date(`${startDate}T${startTime}`);
            const end = new Date(`${endDate}T${endTime}`);
            const diff = Math.max(0, Math.floor((end - start) / 1000));
            document.getElementById('timer-total').textContent = TimerManager.formatTime(diff);
        }
    };
    
    // Event listeners
    modal.querySelectorAll('input').forEach(input => {
        input.addEventListener('change', updateTotal);
    });
    
    modal.querySelectorAll('.quick-button').forEach(btn => {
        btn.addEventListener('click', () => {
            const minutes = parseInt(btn.dataset.minutes);
            const start = new Date(Date.now() + (minutes * 60 * 1000));
            document.getElementById('timer-start-time').value = start.toTimeString().slice(0, 5);
            document.getElementById('timer-start-date').value = start.toISOString().split('T')[0];
            updateTotal();
        });
    });
    
    modal.querySelector('.clear-btn').addEventListener('click', () => {
        TaskStorage.updateTask(taskId, {
            timerStart: null,
            timerElapsed: 0,
            timerRunning: false,
            actualTime: 0
        });
        TimerManager.stopTimer(taskId);
        document.body.removeChild(overlay);
        this.render();
    });
    
    modal.querySelector('.cancel-btn').addEventListener('click', () => {
        document.body.removeChild(overlay);
    });
    
    modal.querySelector('.save-btn').addEventListener('click', () => {
        const startDate = document.getElementById('timer-start-date').value;
        const startTime = document.getElementById('timer-start-time').value;
        const endDate = document.getElementById('timer-end-date').value;
        const endTime = document.getElementById('timer-end-time').value;
        
        if (startDate && startTime && endDate && endTime) {
            const start = new Date(`${startDate}T${startTime}`);
            const end = new Date(`${endDate}T${endTime}`);
            const elapsed = Math.max(0, Math.floor((end - start) / 1000));
            
            TaskStorage.updateTask(taskId, {
                timerStart: null,
                timerElapsed: elapsed,
                timerRunning: false,
                actualTime: elapsed
            });
            
            TimerManager.stopTimer(taskId);
        }
        
        document.body.removeChild(overlay);
        this.render();
    });
    
    // Close on overlay click
    overlay.addEventListener('click', (e) => {
        if (e.target === overlay) {
            document.body.removeChild(overlay);
        }
    });
    
    updateTotal();
}
```

## Step 6: Update the UIManager Constructor
In the `UIManager` constructor, add timer restoration after `this.render()`:

```javascript
constructor() {
    this.taskInput = document.getElementById('taskInput');
    this.taskList = document.getElementById('taskList');
    this.setupEventListeners();
    this.render();
    // Add this line:
    TimerManager.restoreRunningTimers();
}
```

## Test Cases

After implementation, test these scenarios:

1. **Start/Stop Timer**:
   - Click ▶ button - should change to ⏸ and timer starts counting
   - Click ⏸ button - should change to ▶ and timer stops
   - Timer badge should update every second when running

2. **Manual Time Adjustment**:
   - Click timer badge `⏲ --` or `⏲ 1:23:45`
   - Modal should appear with time inputs
   - Test quick buttons (15m ago, 30m ago, etc.)
   - Save should update the timer badge

3. **Multiple Timers**:
   - Start timer on Task A
   - Start timer on Task B
   - Task A should automatically stop when B starts

4. **Persistence**:
   - Start a timer
   - Refresh the page
   - Timer should continue from where it left off

5. **Timer Display**:
   - Under 1 hour: Shows as `0:45` (minutes:seconds)
   - Over 1 hour: Shows as `1h 23m`

## Visual Specifications

### Timer Button
- Position: Left of task title
- Play icon: ▶ (gray #6b7280)
- Pause icon: ⏸ (blue #3b82f6)
- Size: 20x20px
- Hover: Scale 1.1x

### Timer Badge
- Position: Last badge (after hours)
- Icon: ⏲
- Color: Purple (#a855f7) when active
- Empty state: Gray (#6b7280) with dashed border
- Format: `⏲ 1:23:45` or `⏲ --`

### Timer Modal
- Dark background overlay
- Centered modal (400px wide)
- Time/date inputs
- Quick select buttons
- Clear, Cancel, Save actions

## DO NOT
- Change any existing functionality
- Modify colors or styling beyond specified
- Add features not specified
- Use any external libraries
- Change the timer format

## File Structure
This modifies the existing `legal-tasks.html` file. Do NOT create a new file.
