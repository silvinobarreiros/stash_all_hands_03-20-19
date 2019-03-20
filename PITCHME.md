# Functional Patterns In Practice
---

<img src="assets/eugene.jpg" style="height: 85%; width: 85%;"/>
---

## WhoAMI

@ul

- Lead Engineer @ Stash
- Pug Enthusiast
- C -> Java -> C# -> Java -> Scala

@ulend
---

## @pugsnaps on IG
<img src="assets/diesel.jpg" style="height: 40%; width: 40%;"/>
---

## Agenda

@ul

- chat a bit about "fp in rl"
- patterns we follow at Stash
- example use case from our bank product

@ulend

---

## Sooooo What's This In Practice Thing?

@ul

- engineering at a start-up
- balancing monuments and speed
- subset of features
- on-boarding new developers

@ulend

---

## Things we do/don't do here

@ul

- do... 🙅🏻‍♂ `nulls` ️
- do... favor `val` over `var`
- don't... throw exceptions, for the most part
- do... avoid `Unit`
- do... use implicits to help readability, within reason

@ulend

---

## Avoiding nulls

@ul

- Either[Error, Result] 🙌🏼
- Either > Option

@ulend
---

## Throwing Exceptions 👎🏼
@ul

- *cough* Either[Error, Result] *cough*
- function completeness (good for testing)
- buuut not pure

@ulend
---

<img src="https://media.giphy.com/media/12BxzBy3K0lsOs/giphy.gif" style="height: 50%; width: 50%;"/>
---
## End of the world

@ul

- not really the haskell io eow
- more like a REST controller
- status codes + meaningful error messages

@ulend
---

## Example Time

- trace a call through our bank platform to activate a new card
---

## Example Time

end user (phone)
---

## Example Time

end user (phone) --> account service
---

## Example Time

end user (phone) --> account service --> banking partner
---


## Our Stack

@ul

- micro services + akka
- aws
- sttp
- doobie

@ulend
---

## And Some Patterns We Use
---

## Handling IO + Errors

```scala
package object platform {
  type StashResponse[A, B] = Future[Either[A, B]]
  type StashServiceResponse[A] = Future[Either[ErrorResponse, A]]
}
```

@[2](any io operation)
@[3](services with client facing APIs)
---

## When the bad things happen

```json
{
  "errors": [
    {
      "code": 1,
      "namespace": "transferService",
      "message": "Something went wrong with your transfer 😮"
    }
  ]
}
```

@[2](list of errors)
@[3-7]
@[4,5](unique code + domain)
@[6](message for the user 🤗)
---

## Error case class

```scala
case class Error(
  code: Int,
  namespace: String,
  message: String,
  description: Option[String]
)
```
---

## Http w/sttp

@ul

- sttp 😍
- error codes are `Left`s 🙏🏼

@ulend
---

## Our End of the World.. err beginning of the world 

```scala
class UsersController(dependencies: Dependencies) extends LoggedController {
  
  val cardRoute = "users" / userId / "accounts" / accountId / "cards" / cardId

  def routes: Route =
    routeHandler {
      pathPrefix(cardRoute) { (userId, accountId, cardId) =>
        pathPrefix("activate") {
          // PUT /users/:userId/accounts/:accountId/cards/:cardId/activate
          putLoggedAuthorized(userId, accountId)(ActivateRequest.jsonFormat
            activateReq =>
              complete(activate(userId, accountId, cardId, activateReq))
          }
        }
      }
    }
}
```

@[5-16](route handler)
@[12](request handler)
---

## EOW Continued.. 
```scala
private def activate(
  userId: String, accountId: String, cardId: String, activateReq: ActivateReq
): Future[ToResponseMarshallable] = 
  cardHandler.activate(userId, accountId, cardId, activateReqs).map {
    case Left(error) => error.code match {
      case FailedActivationError.code => 
        val activationError = ErrorResponse(error.copy(code = ProviderError.code))
        ToResponseMarshallable(StatusCodes.BadRequest -> activationError)
      case _ => 
        ToResponseMarshallable(StatusCodes.BadRequest -> 
          new ErrorResponse(error))
    }
    case Right(cardActivateResponse) => 
      ToResponseMarshallable(StatusCodes.OK -> cardActivateResponse)
  }
```
---

## Business Time (Card Handler)

```scala
def activate(
  userId: String, accountId: String, cardId: String, activateReq: ActivateReq
): StashResponse[errors.Error, CardActivateResponse] = {
  
  val result = for {
    response <- EitherT(cardService.activate(accountId, cardId, activateReq))
  } yield {

    response.card.activationStatus match {
      case ActivationStatus.Activated =>
        saveCardActivatedEvent(userId, accountId, cardId).foreach {
          case Left(error) => logger.error(s"Failed message")
          case Right(_) => logger.info(s"Successfully saved message")
        }
        response
      
      case _ => Left(Error(
        FailedActivationError.code,
        ErrorHandling.NAMESPACE, 
        FailedActivationError.message
      ))
    }
  }

  result.value
}
```

@[6](make a request to a 3rd party)
@[9-22](check the card status)
@[10-14](siiick activated, save an event to be published later)
@[12](failed 🤷🏻‍♂️)
@[13](succeeded 🤷🏻‍♂️)
@[17-21](card not active, return error)
---

## EitherT + Cats

```scala
def saveCardActivatedEvent(
  userId: String, accountId: String, cardId: String
): StashResponse[Error, DBEvent] = {
  val result = for {
    card        <- EitherT(cardService.getCard(accountId, cardId))
    eventOption = card.activatedDateTime.map(adt => createEvent(adt, card.`type`))
    event       <- EitherT.fromOption[Future](eventOption, Error("error"))
    dbEvent     <- EitherT(eventsRepository.saveEvent(event).convert)
  } yield dbEvent

  result.value
}
```

@[5](get card from 3rd party)
@[6](make our event)
@[7](convert from Option to Either)
@[8](save to db, and convert the error)
@[11](move back to Either)

---

## 3rd party APIs

```scala
trait Http {
  protected def get[A: JsonReader](uri: Uri): StashResponse[A]

  protected def post[A: JsonReader, B: JsonWriter](uri: Uri, body: B): StashResponse[A]
}

trait CardEndpoints extends Http {

  def getCard(accountId: String, cardId: String): StashResponse[CardResponse] = {
    get[CardResponse](
      uri"https://www.coolservice.io/accounts/$accountId/cards/$cardId"
    )
  }

  def activateCard(
    accountId: String, cardInfo: DecryptedCard
  ): StashResponse[ActivateCardResponse] = ???
}

class ThirdPartyClient extends CardEndpoints {
  protected def post[A: JsonReader, B: JsonWriter](uri: Uri, body: B): StashResponse[A] = {
    auth { req =>
      req.post(uri)
        .body(body)
        .response(asJson[A])
        .send()
        .convert
    }
  }
}
```

@[1-5](yo dawg heard you like HTTP requests)
@[2-4](use Future[Either[Error, A]])
@[6-18](implementation time)
@[20-30]
@[27](convert to StashResponse)

---

## Lawls ok but what's that convert function?
---

## Lawls ok but what's that convert function?
<img src="https://media.giphy.com/media/12NUbkX6p4xOO4/giphy.gif" style="height: 50%; width: 50%;"/>
---

## J/K it's just an implicit

```scala
type SttpResponse[A] = Future[Response[Either[String, A]]]

implicit class SttpConverter[A](sttpResponse: SttpResponse[A]) {
  def convert(implicit ec: ExecutionContext): StashResponse[A] = {

    sttpResponse.map { response =>
      response.body match {
        case Right(res) => res.left.map(Error(_))
        case Left(error) =>
          val responseDetails = parseAndExtractField(error, "responseDetails")
          responseDetails.map( 
            details => Left(Error(details.responseDetails))
          ).joinRight
      }
    }
  }
}
```

@[1](sttp alias)
@[3](define implicit class on our sttp response type)
@[4-16](convert method)
@[6-15]
@[7-14]
@[8](2XX + can we parse the response)
@[9-13](not 2XX try to parse the error payload)

---

## Back to the end of the world

```scala
private def activate(
  userId: String, accountId: String, cardId: String, activateReq: ActivateReq
): Future[ToResponseMarshallable] = 
  cardHandler.activate(userId, accountId, cardId, activateReqs).map {
    case Left(error) => error.code match {
      case FailedActivationError.code => 
        val activationError = ErrorResponse(error.copy(code = ProviderError.code))
        ToResponseMarshallable(StatusCodes.BadRequest -> activationError)
      case _ => 
        ToResponseMarshallable(StatusCodes.BadRequest -> new ErrorResponse(error))
    }
    case Right(cardActivateResponse) => 
      ToResponseMarshallable(StatusCodes.OK -> cardActivateResponse)
  }
```
@[4-15]
@[5-11](handle errors)
@[6-8](FailedActivationError explicitly)
@[9-10](all other errors)
@[12-13](sometimes we can have nice things 😮)
---

## Questions

😬
