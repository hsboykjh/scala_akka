# scala_akka

Scheduling service based Akka is running ( Time set in config trigers the Job event )
And User can control the Job start when certain time through the HTTP protocol(GET/POST)


## Server
1. Akka scheduling service
2. Akka HTTP service

### Server init
<pre><code>
object Server extends App {

  val logger = Logger.getLogger(this.getClass)

  logger.info("==1== Connecting to mysql database")
  val mysqlDatabase = Database.forConfig("mysql")

  val actorSystem = ActorSystem("SchedulingSystem")

  val scheduler = new Scheduler(actorSystem, mysqlDatabase)

  logger.info("==2== Scheduling jobs")
  val isDataInit = ConfigFactory.load().getBoolean("akka.isDataInit")
  val scheduleReceiver = scheduler.scheduleJobs(isDataInit)

  val hostname = ConfigFactory.load().getString("akka.http.hostname")
  val port = ConfigFactory.load().getInt("akka.http.port")

  logger.info(s"==3== Scheduling Web Server(HTTP) => http://${hostname}:${port}/")
  val route = new routes(actorSystem, scheduleReceiver)
  val routeInit = route.init(hostname, port)

  // Add shutdown hook to free resources
  sys.addShutdownHook {
    logger.info("==== Shutting down the scheduling server")
  }
}
</code></pre>

### MessageSender class
<pre><code>
class MessageSender(scheduleReceiver: ActorRef) extends Actor with Logger {

  def receive = {
    case message: Job =>
      log.info("MessageSender received Message: "+ message)
      scheduleReceiver ! message
  }
}
</code></pre>

### routes class
<pre><code>
class routes(actorSystem: ActorSystem, scheduleReceiver: ActorRef) extends Logger {
  
  // HTTP routes
  implicit val system = actorSystem
  implicit val materializer = ActorMaterializer()

  def init( hostname: String, port: Int): Unit = {
    val MessageSender = system.actorOf(Props( new MessageSender(scheduleReceiver)), name = "MessageSender")

    val requestHandler: HttpRequest => HttpResponse = {

      case HttpRequest(POST, Uri.Path("/api/jobs/firstJob"), _, _, _) =>
      {
        MessageSender ! firstJob
        HttpResponse(200)
      }
    }

    val bindingFuture = Http().bindAndHandleSync(requestHandler, hostname, port)
  }
}
</code></pre>

### Scheduler class
<pre><code>
class Scheduler(actorSystem: ActorSystem, db: Database) extends Logger {

  private val quartzScheduler = QuartzSchedulerExtension(actorSystem)

  // Schedule to download daily prices and store to database
  def scheduleJobs(isDataInit: Boolean): ActorRef = {
    val driver = MySQLDriver
    val receiver = actorSystem.actorOf(Props(classOf[JobReceiver], driver, db))

    val jobs = Seq(
      firstJob
    )

    jobs.foreach { job =>
      val dateTimeFirstFire = quartzScheduler.schedule(job.toString, receiver, job)
      log.info(s"$job to first fire: $dateTimeFirstFire")
    }
    receiver
  }
}
</code></pre>

### MessageReceiver class
<pre><code>
class MessageReceiver(val scheduleReceiver: ActorRef) extends Actor with Logger {

  def receive = {
    case message: Job =>
      scheduleReceiver ! message
      
    case message: String =>
      log.info(message)
  }
}
</code></pre>

### JobReceiver class
<pre><code>
class JobReceiver(driver: JdbcProfile, db: Database) extends Actor with Logger {
  ...
  def receive = {

    // define job
    case firstJob =>
      log.info("firstJob")
  }
}
</code></pre>
