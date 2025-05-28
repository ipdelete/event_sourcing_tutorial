In this tutorial, we'll build an event sourcing system for a cycling coach AI by implementing this specific workflow:

1. **TrainingPlanCreated** - Athlete preparing for century ride in 16 weeks
2. **TrainingBlockAdded** - 4-week base building phase
3. **WorkoutScheduled** - "60min Zone 2 endurance" for tomorrow
4. **WorkoutCompleted** - Athlete completes the workout (next day)
5. **PerformanceAnalyzed** - AI detects athlete struggling with prescribed power
6. **TrainingLoadAdjusted** - AI reduces next week's intensity by 5%

We'll build this step by step, learning core event sourcing concepts along the way.

## Step 1: Foundation - Events and Event Store

First, let's create our basic event infrastructure:

```python
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import Dict, List, Any, Optional
from uuid import uuid4
import json

@dataclass
class Event:
    """Base event class - immutable record of something that happened"""
    aggregate_id: str
    event_type: str
    timestamp: datetime
    data: Dict[str, Any]
    version: int
    event_id: str = field(default_factory=lambda: str(uuid4()))

class InMemoryEventStore:
    """Simple in-memory event store for learning"""
    
    def __init__(self):
        self.events: List[Event] = []
        self.aggregate_events: Dict[str, List[Event]] = {}
        
    def append_events(self, aggregate_id: str, expected_version: int, events: List[Event]):
        """Append events with optimistic concurrency control"""
        current_version = self.get_current_version(aggregate_id)
        
        if current_version != expected_version:
            raise Exception(f"Concurrency conflict: expected {expected_version}, got {current_version}")
            
        for event in events:
            self.events.append(event)
            if aggregate_id not in self.aggregate_events:
                self.aggregate_events[aggregate_id] = []
            self.aggregate_events[aggregate_id].append(event)
            
    def get_events(self, aggregate_id: str) -> List[Event]:
        """Get all events for an aggregate"""
        return self.aggregate_events.get(aggregate_id, [])
        
    def get_current_version(self, aggregate_id: str) -> int:
        """Get current version (number of events) for aggregate"""
        return len(self.aggregate_events.get(aggregate_id, []))
        
    def get_all_events(self) -> List[Event]:
        """Get all events in chronological order"""
        return sorted(self.events, key=lambda e: e.timestamp)
```

**Key Concepts:**

- **Events are immutable** - Once created, they never change
- **Append-only** - We only add events, never update or delete
- **Versioning** - Each aggregate has a version number to handle concurrency

## Step 2: Training Plan Aggregate

Now let's create our main aggregate that will handle training plan logic:

```python
class TrainingPlan:
    """Aggregate root for training plan domain logic"""
    
    def __init__(self, plan_id: str):
        # Identity
        self.id = plan_id
        self.version = 0
        
        # State (rebuilt from events)
        self.athlete_id: Optional[str] = None
        self.target_event: Optional[str] = None
        self.duration_weeks: Optional[int] = None
        self.training_blocks: List[Dict] = []
        self.scheduled_workouts: Dict[str, Dict] = {}  # date -> workout
        self.completed_workouts: List[Dict] = []
        self.current_training_load: float = 1.0  # multiplier for intensity
        
        # Track uncommitted events
        self.uncommitted_events: List[Event] = []
        
    # Command Methods (public interface)
    
    def create_plan(self, athlete_id: str, target_event: str, duration_weeks: int):
        """Create a new training plan"""
        if self.athlete_id is not None:
            raise Exception("Training plan already exists")
            
        event = Event(
            aggregate_id=self.id,
            event_type="TrainingPlanCreated",
            timestamp=datetime.now(),
            data={
                "athlete_id": athlete_id,
                "target_event": target_event,
                "duration_weeks": duration_weeks
            },
            version=self.version + 1
        )
        
        self._apply_event(event)
        self.uncommitted_events.append(event)
        
    def add_training_block(self, phase_name: str, duration_weeks: int, focus: str):
        """Add a training block to the plan"""
        if not self.athlete_id:
            raise Exception("Cannot add training block to non-existent plan")
            
        event = Event(
            aggregate_id=self.id,
            event_type="TrainingBlockAdded",
            timestamp=datetime.now(),
            data={
                "phase_name": phase_name,
                "duration_weeks": duration_weeks,
                "focus": focus
            },
            version=self.version + 1
        )
        
        self._apply_event(event)
        self.uncommitted_events.append(event)
        
    def schedule_workout(self, date: datetime, workout_type: str, duration_minutes: int, target_zone: str):
        """Schedule a specific workout"""
        date_key = date.strftime("%Y-%m-%d")
        
        if date_key in self.scheduled_workouts:
            raise Exception(f"Workout already scheduled for {date_key}")
            
        event = Event(
            aggregate_id=self.id,
            event_type="WorkoutScheduled",
            timestamp=datetime.now(),
            data={
                "date": date_key,
                "workout_type": workout_type,
                "duration_minutes": duration_minutes,
                "target_zone": target_zone
            },
            version=self.version + 1
        )
        
        self._apply_event(event)
        self.uncommitted_events.append(event)
        
    def complete_workout(self, date: datetime, actual_power: int, perceived_effort: int, notes: str = ""):
        """Record completion of a workout"""
        date_key = date.strftime("%Y-%m-%d")
        
        if date_key not in self.scheduled_workouts:
            raise Exception(f"No workout scheduled for {date_key}")
            
        event = Event(
            aggregate_id=self.id,
            event_type="WorkoutCompleted",
            timestamp=datetime.now(),
            data={
                "date": date_key,
                "actual_power": actual_power,
                "perceived_effort": perceived_effort,
                "notes": notes
            },
            version=self.version + 1
        )
        
        self._apply_event(event)
        self.uncommitted_events.append(event)
        
    def analyze_performance(self, analysis_result: str, struggling_areas: List[str]):
        """AI analyzes recent performance"""
        event = Event(
            aggregate_id=self.id,
            event_type="PerformanceAnalyzed",
            timestamp=datetime.now(),
            data={
                "analysis_result": analysis_result,
                "struggling_areas": struggling_areas
            },
            version=self.version + 1
        )
        
        self._apply_event(event)
        self.uncommitted_events.append(event)
        
    def adjust_training_load(self, adjustment_percentage: float, reason: str):
        """Adjust future training intensity"""
        new_load = self.current_training_load * (1 + adjustment_percentage / 100)
        
        event = Event(
            aggregate_id=self.id,
            event_type="TrainingLoadAdjusted",
            timestamp=datetime.now(),
            data={
                "old_training_load": self.current_training_load,
                "new_training_load": new_load,
                "adjustment_percentage": adjustment_percentage,
                "reason": reason
            },
            version=self.version + 1
        )
        
        self._apply_event(event)
        self.uncommitted_events.append(event)
        
    # Event Application (private)
    
    def _apply_event(self, event: Event):
        """Apply an event to update aggregate state"""
        if event.event_type == "TrainingPlanCreated":
            self.athlete_id = event.data["athlete_id"]
            self.target_event = event.data["target_event"]
            self.duration_weeks = event.data["duration_weeks"]
            
        elif event.event_type == "TrainingBlockAdded":
            self.training_blocks.append({
                "phase_name": event.data["phase_name"],
                "duration_weeks": event.data["duration_weeks"],
                "focus": event.data["focus"]
            })
            
        elif event.event_type == "WorkoutScheduled":
            self.scheduled_workouts[event.data["date"]] = {
                "workout_type": event.data["workout_type"],
                "duration_minutes": event.data["duration_minutes"],
                "target_zone": event.data["target_zone"]
            }
            
        elif event.event_type == "WorkoutCompleted":
            self.completed_workouts.append({
                "date": event.data["date"],
                "actual_power": event.data["actual_power"],
                "perceived_effort": event.data["perceived_effort"],
                "notes": event.data["notes"]
            })
            
        elif event.event_type == "PerformanceAnalyzed":
            # Store analysis in state if needed
            pass
            
        elif event.event_type == "TrainingLoadAdjusted":
            self.current_training_load = event.data["new_training_load"]
            
        self.version = event.version
        
    def mark_events_as_committed(self):
        """Clear uncommitted events after saving"""
        self.uncommitted_events = []
        
    @classmethod
    def from_events(cls, plan_id: str, events: List[Event]):
        """Rebuild aggregate from event history"""
        plan = cls(plan_id)
        for event in events:
            plan._apply_event(event)
        return plan
```

**Key Concepts:**

- **Aggregate encapsulates business logic** - All rules and validations are here
- **Commands generate events** - Public methods create events, don't directly modify state
- **Events update state** - Only `_apply_event` modifies the aggregate's state
- **Rebuilding from events** - `from_events` replays the entire history

## Step 3: Repository Pattern

Now let's create a repository to save and load our aggregates:

```python
class TrainingPlanRepository:
    """Repository for loading and saving training plans"""
    
    def __init__(self, event_store: InMemoryEventStore):
        self.event_store = event_store
        
    def get(self, plan_id: str) -> TrainingPlan:
        """Load aggregate from event store"""
        events = self.event_store.get_events(plan_id)
        return TrainingPlan.from_events(plan_id, events)
        
    def save(self, plan: TrainingPlan):
        """Save uncommitted events to store"""
        if plan.uncommitted_events:
            expected_version = plan.version - len(plan.uncommitted_events)
            self.event_store.append_events(
                plan.id,
                expected_version,
                plan.uncommitted_events
            )
            plan.mark_events_as_committed()
```

**Key Concepts:**

- **Repository pattern** - Abstracts storage details from domain logic
- **Optimistic concurrency** - Uses expected version to detect conflicts
- **Uncommitted events** - Tracks which events need to be saved

## Step 4: Working Through Our Example

Now let's implement our complete workflow:

```python
def run_cycling_coach_workflow():
    """Run through our complete example event flow"""
    
    # Setup
    event_store = InMemoryEventStore()
    repository = TrainingPlanRepository(event_store)
    plan_id = "plan-001"
    
    print("🚴 Cycling Coach AI Event Sourcing Demo\n")
    
    # Step 1: TrainingPlanCreated
    print("Step 1: Creating training plan for century ride...")
    plan = TrainingPlan(plan_id)
    plan.create_plan(
        athlete_id="athlete-123",
        target_event="Century Ride (100 miles)",
        duration_weeks=16
    )
    repository.save(plan)
    print(f"✅ Plan created for athlete-123, targeting century ride in 16 weeks")
    print(f"   Current version: {plan.version}\n")
    
    # Step 2: TrainingBlockAdded
    print("Step 2: Adding base building training block...")
    plan = repository.get(plan_id)  # Reload from events
    plan.add_training_block(
        phase_name="Base Building",
        duration_weeks=4,
        focus="Aerobic capacity and endurance"
    )
    repository.save(plan)
    print(f"✅ Added 4-week base building phase")
    print(f"   Current version: {plan.version}")
    print(f"   Training blocks: {len(plan.training_blocks)}\n")
    
    # Step 3: WorkoutScheduled
    print("Step 3: Scheduling tomorrow's workout...")
    plan = repository.get(plan_id)
    tomorrow = datetime.now() + timedelta(days=1)
    plan.schedule_workout(
        date=tomorrow,
        workout_type="Endurance",
        duration_minutes=60,
        target_zone="Zone 2"
    )
    repository.save(plan)
    print(f"✅ Scheduled 60min Zone 2 endurance workout for {tomorrow.strftime('%Y-%m-%d')}")
    print(f"   Current version: {plan.version}")
    print(f"   Scheduled workouts: {len(plan.scheduled_workouts)}\n")
    
    # Step 4: WorkoutCompleted
    print("Step 4: Athlete completes the workout...")
    plan = repository.get(plan_id)
    plan.complete_workout(
        date=tomorrow,
        actual_power=180,  # watts - lower than expected for Zone 2
        perceived_effort=8,  # out of 10 - high effort
        notes="Struggled to maintain target power, legs felt heavy"
    )
    repository.save(plan)
    print(f"✅ Workout completed with 180W avg power, RPE 8/10")
    print(f"   Current version: {plan.version}")
    print(f"   Completed workouts: {len(plan.completed_workouts)}\n")
    
    # Step 5: PerformanceAnalyzed  
    print("Step 5: AI analyzes performance...")
    plan = repository.get(plan_id)
    plan.analyze_performance(
        analysis_result="Athlete showing signs of fatigue and power decline",
        struggling_areas=["power_output", "perceived_effort_high"]
    )
    repository.save(plan)
    print(f"✅ Performance analyzed - detected struggling with prescribed power")
    print(f"   Current version: {plan.version}\n")
    
    # Step 6: TrainingLoadAdjusted
    print("Step 6: AI adjusts training load...")
    plan = repository.get(plan_id)
    plan.adjust_training_load(
        adjustment_percentage=-5,  # Reduce by 5%
        reason="Performance analysis indicates athlete struggling with current load"
    )
    repository.save(plan)
    print(f"✅ Training load reduced by 5% (new load: {plan.current_training_load:.2f})")
    print(f"   Current version: {plan.version}\n")
    
    # Show complete event history
    print("📜 Complete Event History:")
    print("=" * 50)
    all_events = event_store.get_events(plan_id)
    for i, event in enumerate(all_events, 1):
        print(f"{i}. {event.event_type} (v{event.version})")
        print(f"   Time: {event.timestamp.strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"   Data: {json.dumps(event.data, indent=6)}")
        print()
    
    return plan, event_store

# Run the demo
if __name__ == "__main__":
    final_plan, store = run_cycling_coach_workflow()
```

## Step 5: Key Benefits We've Achieved

Let's add some utility functions to demonstrate the power of event sourcing:

```python
def demonstrate_event_sourcing_benefits(plan: TrainingPlan, store: InMemoryEventStore):
    """Show off what event sourcing gives us"""
    
    print("🎯 Event Sourcing Benefits Demonstrated:\n")
    
    # 1. Complete Audit Trail
    print("1. Complete Audit Trail:")
    events = store.get_events(plan.id)
    for event in events:
        print(f"   {event.timestamp.strftime('%H:%M:%S')} - {event.event_type}")
    print()
    
    # 2. Time Travel - Rebuild state at any point
    print("2. Time Travel - State after step 3 (before workout):")
    events_up_to_step3 = events[:3]  # First 3 events
    historical_plan = TrainingPlan.from_events(plan.id, events_up_to_step3)
    print(f"   Scheduled workouts: {len(historical_plan.scheduled_workouts)}")
    print(f"   Completed workouts: {len(historical_plan.completed_workouts)}")
    print(f"   Training load: {historical_plan.current_training_load}")
    print()
    
    # 3. Event Replay for Different Scenarios
    print("3. What-if Analysis - Different training load adjustment:")
    alt_plan = TrainingPlan.from_events(plan.id, events[:-1])  # All but last event
    alt_plan.adjust_training_load(-10, "Alternative: more conservative reduction")
    print(f"   Original adjustment: 5% reduction")
    print(f"   Alternative: 10% reduction would result in load: {alt_plan.current_training_load:.2f}")
    print()
    
    # 4. Temporal Queries
    print("4. Temporal Queries:")
    performance_events = [e for e in events if "Performance" in e.event_type or "Completed" in e.event_type]
    print(f"   Performance-related events: {len(performance_events)}")
    
    load_adjustments = [e for e in events if e.event_type == "TrainingLoadAdjusted"]
    if load_adjustments:
        print(f"   Training load adjustments: {len(load_adjustments)}")
        latest_adjustment = load_adjustments[-1]
        print(f"   Latest adjustment reason: {latest_adjustment.data['reason']}")

# Add this to the main workflow
if __name__ == "__main__":
    final_plan, store = run_cycling_coach_workflow()
    demonstrate_event_sourcing_benefits(final_plan, store)
```

## Next Steps & Advanced Concepts

This tutorial covers the fundamentals, but here are areas to explore next:

### Event Versioning

```python
# Handle schema evolution
@dataclass
class WorkoutScheduledV2:
    date: datetime
    workout_type: str
    duration_minutes: int
    target_zones: Dict[str, int]  # Multiple zones instead of single string
    weather_conditions: Optional[str] = None  # New field
```

### Projections (Read Models)

```python
class AthleteTrainingStats:
    """Read model optimized for dashboard queries"""
    def __init__(self):
        self.total_workouts = 0
        self.avg_power = 0
        self.training_load_history = []
        
    def handle_event(self, event: Event):
        if event.event_type == "WorkoutCompleted":
            self.total_workouts += 1
            # Update stats...
```

### Event Bus for Reactive Behavior

```python
class EventBus:
    def __init__(self):
        self.handlers = []
        
    def publish(self, events: List[Event]):
        for event in events:
            for handler in self.handlers:
                handler.handle(event)
```

### Snapshots for Performance

```python
def create_snapshot(plan: TrainingPlan) -> Dict:
    """Save current state to avoid replaying all events"""
    return {
        "athlete_id": plan.athlete_id,
        "training_blocks": plan.training_blocks,
        "current_training_load": plan.current_training_load,
        # ... other state
    }
```

This tutorial demonstrates the core concepts of event sourcing through a practical cycling coach AI domain. The key insight is that by modeling what happened (events) rather than current state, we get a rich, auditable, and flexible system that can adapt and learn over time.