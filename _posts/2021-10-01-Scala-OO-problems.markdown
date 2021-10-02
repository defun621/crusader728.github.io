---
layout: post
title:  Solving OOP design interview problems with Scala and FP (Parking lot)
date:   2021-10-01 18:04:30 -0700
categories: 
    - Programming
    - Scala
---

It's been a while since I practise my Scala skill last time. 

I think it might be interesting as a warmup if I try to solve those OOP design interview problem with Scala and FP building blocks.

The problem I'm looking at today is to design a parking lot.

Here is the problem definition.

> n levels, each level has m rows of spots and each row has k spots.
> So each level has m x k spots.
> 
> The parking lot can park motorcycles, cars and buses.
> 
> The parking lot has motorcycle spots, compact spots, and large spots.
> 
> Each row, motorcycle spots id is in range [0,k/4)(0 is included, k/4 is not included), compact spots id is in range [k/4,k/4\*3) and large spots id is in range [k/4\*3,k).
> 
> A motorcycle can park in any spot.
> A car park in single compact spot or large spot.
> A bus can park in five large spots that are consecutive and within same row. it can not park in small spots

First is the definition of *Vehicle*: 

```scala
sealed abstract class Vehicle extends Product with Serializable
case class Motorcycle(licensePlate: String) extends Vehicle
case class Car(licensePlate: String) extends Vehicle
case class Bus(licensePlate: String) extends Vehicle
```

, and *ParkingLot*:

```scala
sealed abstract class ParkingSpot extends Product with Serializable {
    def id: Int
}
case class Motor(id: Int) extends ParkingSpot
case class Compact(id: Int) extends ParkingSpot
case class Large(id: Int) extends ParkingSpot

case class Row(spots: Vector[ParkingSpot])
case class Level(rows: Vector[Row])
case class ParkingLot(levels: Vector[Level])
```

Secondly, Let's define the companion object of ParkingLot:

```scala
object ParkingLot {
    def apply(n: Int, m: Int, k: Int): ParkingLot = ParkingLot {
        (0 until n).toVector.map(i => Level(m, k, m * k * i))
    }
}

object Level {
    def apply(m: Int, k: Int, idBase: Int): Level = Level {
        (0 until m).toVector.map(j => Row(k, idBase + j * k))
    }
}

object Row {
    def apply(k: Int, idBase: Int): Row = Row {
        (0 until k).toVector.map {
            case x if x >= 0 && x < k / 4 => Motor(idBase + x)
            case x if x >= k / 4 && x < k / 4 * 3 => Compact(idBase + x)
            case x@_ => Large(idBase + x)
        }
    }
}
```
, so that we could create a ParkingLot with *ParkingLot(1, 1, 20)*.

Each time a vehicle parks in a parking lot, a spot is taken. Also the state of the parking lot changes. Therefore, we need a way to track the takens spots. This a typical State Monad use case.

Basically the state we care about is the occupied spots. So here is the case class for tracking occupied spots and the emitted data:
```scala
case class ParkingLotOccupied(occupied: Set[Int])
case class SpotTaken(pos: List[Int])
```
, and We can give a type alias for the state monad we use:
```scala
type ParkingLotState[A] = State[ParkingLotOccupied, A]
```

With all the basic building blocks in hand, Let's define the behavior, which are *park* and *unpark*.
```scala
trait ParkingVehicleService[V <: Vehicle] {
    def park(v: V, parkingLot: ParkingLot): ParkingLotState[SpotTaken]
}
trait UnparkingVehicleService {
    def unpark(v: Vehicle, ticket: SpotTaken): ParkingLotState[Unit]
}
```

Let's tackle the problem from easy to hard. Unparking is the easier one because all we need to do is to remove the spots from the data structure. So the simplest implmenetation should be:
```scala
trait UnparkingVehicleService {
    def unpark(v: Vehicle, ticket: SpotTaken): ParkingLotState[Unit] = State[ParkingLotOccupied, Unit](occupid => {
        (ParkingLotOccupied(occupid.occupied -- ticket.pos.toSet), ()）
    })
}
```
Now let's move on to the ParkingVehicleService. 

To find a valid spot, we need to find ways to iterate through all spots.
For motorcycle and car, the iterator is to go through every spot. 
For bus, it's sliding window of 5 spots.
```scala
implicit class toIteratorOps(val parkingLot: ParkingLot) extends AnyVal {
    def toSpotIterator: Iterator[ParkingSpot] =
        (for {
            l <- parkingLot.levels
            r <- l.rows
            p <- r.spots
        } yield p).iterator

    def to5SpotsIterator: Iterator[Vector[ParkingSpot]] =
        (for {
            l <- parkingLot.levels
            r <- l.rows
            w <- r.spots.sliding(5)
        } yield w).iterator
}
```
With the iterators we have at hands, we can define the parking vehicle services.
```scala
final class MotorcycleParkingService extends ParkingVehicleService[Motorcycle] {
    override def park(v: Motorcycle, parkingLot: ParkingLot): ParkingLotState[SpotTaken] = State(s => {
        val ticket = parkingLot.toSpotIterator.find(spot => !s.occupied(spot.id))
        ticket.fold((s, SpotTaken(List.empty)))(spot => (ParkingLotOccupied(s.occupied + spot.id), SpotTaken(List(spot.id))))
    })
}

final class CarParkingService extends ParkingVehicleService[Car] {
    override def park(v: Car, parkingLot: ParkingLot): ParkingLotState[SpotTaken] = State(s => {
        val spotTaken = parkingLot
        .toSpotIterator
        .find {
            case Motor(_) => false
            case Compact(id) => !s.occupied(id)
            case Large(id) => !s.occupied(id)
        }
        spotTaken.fold((s, SpotTaken(List.empty)))(spot => (ParkingLotOccupied(s.occupied + spot.id), SpotTaken(List(spot.id))))
    })
}

final class BusParkingService extends ParkingVehicleService[Bus] {
    override def park(v: Bus, parkingLot: ParkingLot): ParkingLotState[SpotTaken] = State(s => {
        val spotsTaken = parkingLot
        .to5SpotsIterator
        .find {
            case x if x.size != 5 => false
            case x if x.forall {
            case Large(_) => true
            case _ => false
            } => true
            case _ => false
        }
        spotsTaken.fold((s, SpotTaken(List.empty)))(v => (ParkingLotOccupied(s.occupied ++ v.map(_.id).toSet), SpotTaken(v.toList.map(_.id))))
    })
}
```

In order to use the service in a more convenient way, we define a few extends methods for motorcycle, car, and bus.
```scala
object ParkingService {
    implicit val motorcycleParking: ParkingVehicleService[Motorcycle] = new MotorcycleParkingService
    implicit val busParkingService: ParkingVehicleService[Bus] = new BusParkingService
    implicit val carParkingService: ParkingVehicleService[Car] = new CarParkingService

    implicit class CarParking(val car: Car) extends AnyVal {
        def park(p: ParkingLot): ParkingLotState[SpotTaken] = {
            implicitly[ParkingVehicleService[Car]].park(car, p)
        }
    }
    implicit class MotorcyclePark(val motorcycle: Motorcycle) extends AnyVal {
        def park(p: ParkingLot): ParkingLotState[SpotTaken] = {
            implicitly[ParkingVehicleService[Motorcycle]].park(motorcycle, p)
        }
    }
    implicit class BusPark(val bus: Bus) extends AnyVal {
        def park(p: ParkingLot): ParkingLotState[SpotTaken] = {
            implicitly[ParkingVehicleService[Bus]].park(bus, p)
        }
    }
}

object UnparkingVehicleService {
    private val carUnparkService = new UnparkingVehicleService {}
    def unpark[V <: Vehicle](v: Vehicle, t: SpotTaken): ParkingLotState[Unit] = carUnparkService.unpark(v, t)
    implicit class UnparkVehicleServiceOps[V <: Vehicle](val v: V) extends AnyVal {
        def unpark(ticket: SpotTaken): ParkingLotState[Unit] = UnparkingVehicleService.unpark(v, ticket)
    }
}
```

In the end, we can write the program like:
```scala
for {
      ticketA <- carA.park(parkingLot)
      ticketB <- carB.park(parkingLot)
      ticketC <- carC.park(parkingLot)
      ticketD <- carD.park(parkingLot)
      _ <- carC.unpark(ticketC)
      ticketDD <- carD.park(parkingLot)
      ticketE <- motorcycle.park(parkingLot)
} yield ()
```
