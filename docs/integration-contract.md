# DC–CT integration contract

Version 0.1.0 | Proposed for agreement

## Scope and responsibilities

Staging and loading are the primary tasks of DCs. Truck location tracking and determination of handling durations is done by CTs. All DC communication is with CTs. CTs relay relevant information to the executing TC, so DC never calls a TC. Tendering and consolidation (i.e., selecting the cheapest transport company for a given order) is managed by the CTs.

## Service catalogue

| Message (DC perspective) | Sender → receiver | Transport         | Trigger                                   |
| ------------------------ | ----------------- | ----------------- | ----------------------------------------- |
| OrderCreated             | CT → DC           | REST POST /orders | Assigned truck and expected arrival known |
| LoadingScheduleCreated   | DC → CT           | REST response     | Initial schedule calculated               |
| LoadingScheduleUpdated   | CT → DC           | RabbitMQ          | Expected arrival changes                  |
| TruckETAUpdated          | DC → CT           | RabbitMQ          | Planned dock/start/departure changes      |
| TruckArrived             | CT → DC           | RabbitMQ          | Truck physically reaches DC               |
| LoadingCompleted         | DC → CT           | RabbitMQ          | Loading actually completes                |

## Lifecycle from DC perspective

1. CT sends the order with staging and loading durations in `POST /orders`. DC uses those durations to plan staging and reserve a dock slot, persists the initial schedule, and returns HTTP 200.
2. CT forwards the schedule (dock and time) to the relevant TC. DC stages goods in advance when feasible to reduce truck queue waiting time.
3. CT publishes ETA revisions and DC recalculates schedules as needed. DC publishes changed loading schedules (e.g., changed dock or changed time). CT forwards latest schedule to TC.
4. CT publishes actual arrival. DC loads only when truck is present, staging complete, and a dock is available. If actual arrival changes dock availability or loading plans, DC may reschedule the arriving order and other affected orders. DC publishes one schedule update per changed order, using that order’s `order_id`.
5. DC publishes confirmation after loading completes. The CT may then authorize onward movement, but CT must wait for all orders assigned to a truck at this DC before departure in case of consolidated orders (multiple orders in one truck).

> [!NOTE]
> Note that loading may only begin once the truck's actual arrival has been confirmed, regardless of its estimated arrival time (ETA). In our simulation, we only care about the 'happy' flow, and not the 'sad' flow, meaning we expect every operation to succeed. This keeps us focused on the main task: showcasing business process integration, not full realism.

## General rules & using standard units

Use of standard units:

- Event times in REST and RabbitMQ use ISO 8601 UTC strings at whole-second precision (`YYYY-MM-DDTHH:mm:ssZ`, e.g., `2026-11-28T10:02:00Z`).
- Durations use integer seconds, not timestamp strings.
- Quantity fields use shared standard quantity units (QU).
- Cost fields use shared standard cost units (CU).
- Distance fields use shared standard distance units (DU).
- Duration, quantity and distance values are non-negative integers. Cost values may have up to two decimal places.
- Round calculated durations up to the nearest whole second.

**Simulation timing.** Schedule simulated events seconds or minutes apart, not hours or days apart, so we can watch the business process integrations run end to end in real time. For example, an order sent at `2026-11-28T10:02:00Z` could have an ETA of `2026-11-28T10:03:00Z`, one real minute later. These are ordinary UTC date-time values: the CT, DC and TC use synchronized system clocks, with no separate simulation clock or shared start time.

### Rules for standard units

#### Staging and loading

The case description requires the CT to determine handling times. The CT includes both durations in the initial `new_order` payload. We propose the following deterministic rules.

1. Staging a quantity $q$ of goods takes $t_s = \lceil q / 2 \rceil$ seconds. E.g., an order of $q = 100$ QU takes 50 seconds to stage.
2. Loading a quantity $q$ of goods takes $t_l = \lceil q / 5 \rceil$ seconds. E.g., an order of $q = 100$ QU takes 20 seconds to load.
3. Staging may begin before a truck arrives at the DC. Loading may only begin once the goods have been staged, the truck has arrived, and a loading dock is available.
4. Each loading dock accommodates a maximum of one truck at a time. Each DC has 3 loading docks.

#### Trucks and transportation

5. A truck can carry a maximum of 100 QU.
6. A truck carrying exactly 100 QU is considered a full truckload (FTL). A truck carrying less than 100 QU of cargo is considered a less-than-truckload (LTL).
7. Multiple orders may be transported by the same truck, provided their combined quantity does not exceed the truck's capacity. Individual orders may not be split across trucks.
8. Each TC initially operates 5 trucks.
9. Trucks travel at a constant speed of 5 DU per second, regardless of their load.
10. Unloading a quantity $q$ of goods at the customer takes $t_u = \lceil q / 5 \rceil$ seconds.
11. Trucks cannot travel while being loaded or unloaded and cannot execute multiple journeys simultaneously. The CT is responsible for tracking their locations and movements.

#### Costs and revenue

> [!WARNING]
> The rules below are applicable to TCs and CTs, not DCs. Since this integration contract is written from the DC perspective, CTs and TCs are expected to discuss and finalize these rules among themselves. Note that CTs and DCs do NOT have money, nor do they earn it. CUs are ONLY relevant for selecting the cheapest TC to transport (an) order(s). This is an abstraction, we do not aim to mimic the real world.

12. Operating a truck costs 1 CU per DU travelled, regardless of its load. Both loaded and empty journeys incur this cost.
13. A truck incurs a cost of 0.5 CU per second while waiting, loading or unloading.
14. TCs deterministically calculate their own bids based on expected costs, and do not deviate from these rules to gain an unfair advantage over other TCs.

## Field clarifications

- `arrival_time_at_dc` is the latest ETA (Estimated Time of Arrival), not actual arrival.
- `actual_arrival_time_at_dc` is a confirmed physical arrival.
- `start_loading_time` is estimated as well, not actual.
- `estimated_departure_time` must be no earlier than `start_loading_time` plus `loading_duration` seconds. For example, loading starting at `2026-11-28T10:02:05Z` with a duration of `20` gives an earliest departure of `2026-11-28T10:02:25Z`. `dock_id` identifies the planned dock.
- `staging_duration` and `loading_duration` are required integer durations in seconds. `staging_duration` may be zero, and `loading_duration` must be positive.

## Communication rules

### REST

Use HTTP 400 for invalid requests, 409 for conflicts and 503 for temporary unavailability. E.g., repeated new order requests with the same order ID return HTTP 409.

## RabbitMQ

_TBD_
