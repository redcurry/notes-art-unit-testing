Using stubs to break dependencies
---------------------------------

**Example:**

LogAnalyzer.IsValidLogFileName(string fileName)
reads through the configuration file given
and returns true if configuration says extension is supported.

Option 1: Dependency injection through constructor
..................................................

- Write an interface to abstract file system::

    public interface IExtensionManager
    {
        bool IsValid(string fileName);
    }

- Write code that uses the interface::

    public class LogAnalyzer
    {
        private IExtensionManager;

        // Use dependency injection through constructor
        public LogAnalyzer(IExtensionManager mgr)
        {
            manager = mgr;
        }

        public bool IsValidLogFileName(string fileName)
        {
            return manager.IsValid(fileName);
        }
    }

- Write the real extension manager::

    public class FileExtensionManager : IExtensionManager
    {
        public bool IsValid(string fileName)
        {
            ...
        }
    }

- Write the fake extension manager::

    internal class FakeExtensionManager : IExtensionManager
    {
        public bool WillBeValid = true;

        public bool IsValid(string fileName)
        {
            return WillBeValid;
        }
    }

- Write the test(s)::

    [TestFixture]
    public class LogAnalyzerTests
    {
        [Test]
        public void IsValidFileName_NameSupportedExtension_ReturnsTrue()
        {
            FakeExtensionManager fakeManager = new FakeExtensionManager();
            fakeManager.WillBeValid = true;

            LogAnalyzer log = new LogAnalyzer(fakeManager);

            bool result = log.IsValidLogFileName("short.ext");
            Assert.True(result);
        }
    }

Option 2: Dependency injection through property
...............................................

- Alternatively, use properties to set the extension manager::

    public class LogAnalyzer
    {
        public IExtensionManager Manager { get; set; }

        // Use a default extension manager
        public LogAnalyzer()
        {
            Manager = new FileExtensionManager();
        }

        public bool IsValidlogFileName(string fileName)
        {
            return Manager.IsValid(fileName);
        }
    }

    [TestFixture]
    public class LogAnalyzerTests
    {
        [Test]
        public void IsValidFileName_NameSupportedExtension_ReturnsTrue()
        {
            FakeExtensionManager fakeManager = new FakeExtensionManager();
            fakeManager.WillBeValid = true;

            LogAnalyzer log = new LogAnalyzer();
            log.Manager = fakeManager;

            bool result = log.IsValidLogFileName("short.ext");
            Assert.True(result);
        }
    }

Option 3: Factory class
.......................

- Two main options within factory class:

    1. Factory class (static) returns fake by setting a property::

        class ExtensionManagerFactory
        {
            private static IExtensionManager manager = null;

            public static IExtensionManager Create()
            {
                if (manager != null)
                    return manager;
                else
                    return new FileExtensionManager();
            }

            public static void SetManager(IExtensionManager mgr)
            {
                manager = mgr;
            }
        }

        public class LogAnalyzer
        {
            private IExtensionManager manager;

            public LogAnalyzer()
            {
                manager = ExtensionManagerFactory.Create();
            }

            public bool IsValidLogFileName(string fileName)
            {
                return manager.IsValid(fileName);
            }
        }

        [TestFixture]
        public class LogAnalyzerTests
        {
            [Test]
            public void IsValidFileName_NameSupportedExtension_ReturnsTrue()
            {
                FakeExtensionManager fakeManager = new FakeExtensionManager();
                fakeManager.WillBeValid = true;
                ExtensionManagerFactory.SetManager(fakeManager);

                LogAnalyzer log = new LogAnalyzer();

                bool result = log.IsValidLogFileName("short.ext");
                Assert.True(result);
            }
        }

    2. Use a fake factory (my own example)::

        public abstract class ExtensionManagerFactory
        {
            public IExtensionManager Create();
        }

        public class FileExtensionManagerFactory : ExtensionManagerFactory
        {
            public override IExtensionManager Create()
            {
                retun new FileExtensionManager();
            }
        }

        public class FakeExtensionManagerFactory : ExtensionManagerFactory
        {
            public override IExtensionManager Create()
            {
                return new FakeExtensionManager();
            }
        }

        public class LogAnalyzer
        {
            private IExtensionManager manager;

            public LogAnalyzer(ExtensionManagerFactory factory)
            {
                manager = factory.Create();
            }

            public bool IsValidLogFileName(string fileName)
            {
                return manager.IsValid(fileName);
            }
        }

        [TestFixture]
        public class LogAnalyzerTests
        {
            [Test]
            public void IsValidFileName_NameSupportedExtension_ReturnsTrue()
            {
                var factory = new FakeExtensionManagerFactory();
                LogAnalyzer log = new LogAnalyzer(factory);

                bool result = log.IsValidLogFileName("short.ext");
                Assert.True(result);
            }
        }

    Me: The factory class could be made to return other fake classes
    that are used throughout the program.

Option 4: Factory method
........................

- Write the class with a virtual method (or virtual property?)::

    public class LogAnalyzer
    {
        public bool IsValidLogFileName(string fileName)
        {
            return GetManager().IsValid(fileName);
        }

        protected virtual IExtensionManager GetManager()
        {
            return new FileExtensionManager();    // default
        }
    }

    class TestableLogAnalyzer : LogAnalyzer
    {
        public IExtensionManager Manager;

        public TestableLogAnalyzer(IExtensionManager mgr)
        {
            Manager = mgr;
        }

        protected override IExtensionManager GetManager()
        {
            return Manager;
        }
    }

    [TestFixture]
    public class LogAnalyzerTests
    {
        [Test]
        public void IsValidFileName_NameSupportedExtension_ReturnsTrue()
        {
            FakeExtensionManager stub = new FakeExtensionManager();
            stub.WillBeValid = true;

            TestableLogAnalyzer log = new TestableLogAnalyzer(stub);

            bool result = log.IsValidLogFileName("short.ext");
            Assert.True(result);
        }
    }

Me: I like this method the least because you're not testing the actual class.
Also, it seems inconvenient to constanly use GetManager(),
although using a virtual property could solve the problem.

Option 5: Virtual methods
.........................

Use a virtual method for the method to actually test,
then override that method to return whatever is needed by the test::

    public class LogAnalyzer
    {
        public bool IsValidLogFileName(string fileName)
        {
            return IsValid(fileName);
        }

        protected virtual bool IsValid(string fileName)
        {
            FileExtensionmanager mgr = new FileExtensionManager();
            return mgr.IsValid(fileName);
        }
    }

    class TestableLoganalyzer : LogAnalyzer
    {
        public bool IsSupported;

        protected override bool IsValid(fileName)
        {
            return IsSupported;
        }
    }

    [TestFixture]
    public class LogAnalyzerTests
    {
        [Test]
        public void IsValidFileName_NameSupportedExtension_ReturnsTrue()
        {
            TestableLogAnalyzer log = new TestableLogAnalyzer(stub);
            log.IsSupported = true;

            bool result = log.IsValidLogFileName("short.ext");
            Assert.True(result);
        }
    }

Mocks
-----

* The basic difference between stubs and mocks is that
  stubs can't fail tests, mocks can.

* Create a mock by first creating an interface (as with a stub)::

    public interface IWebService
    {
        void LogError(string message);
    }

* Implement the mock::

    public class FakeWebService : IWebService
    {
        public string LastError;

        public void LogError(string message)
        {
            LastError = message;
        }
    }

* Use it in a test::

    [Test]
    public void Analyze_TooShortFileName_CallsWebService()
    {
        FakeWebService mockService = new FakeWebService();
        LogAnalyzer log = new LogAnalyzer(mockService);

        log.Analyze("abc.txt");

        StringAssert.Contains("Filename too short:abc.ext",
            mockService.LastError);
    }

* In the example on p. 81, the web service is a stub
  because we need it to throw an exception (not test the web service),
  so that we test whether the log analyzer e-mails the log.
  The e-mail service is a mock because we need to test
  whether the e-mail was sent.

Mocking frameworks
------------------

* Using NSubstitute::

    [Test]
    public void Analyze_tooShortFileName_CallLogger()
    {
        ILogger logger = Substitute.For<ILogger>();
        LogAnalyzer analyzer = new LogAnalyzer(logger);

        analyzer.MinNameLength = 6;
        analyzer.Analyze("a.txt");

        logger.Received().LogError("Filename too short: a.txt");
    }

* `Received()` is used to test whether `LogError` was called on `logger`.

* Make a fake object always return a specific value::

    // Create fake object
    IFileNameRules fakeRules = Substitute.For<IFileNameRules>();

    // With the exact parameter given
    fakeRules.IsValidLogFileName("strict.txt").Returns(true);

    // With any parameter
    fakeRules.IsValidLogFileName(Arg.Any<String>()).Returns(true);

    // Assertion
    Assert.IsTrue(fakeRules.IsValidLogFileName("strict.txt"));

* Simulate an exception::

    fakeRules.When(x => x.IsValidLogFileName(Arg.Any<String>()))
             .Do(context => { throw new Exception("fake exception"); });

    Assert.Throws<Exception>(() => fakeRules.IsValidLogFileName("anything"));

  Me: How is this useful? If you're setting up IsValidLogFileName
  to throw an exception, then why are you testing that it does?
  The point is not to test whether the isolation framework works.

* A more complex example where the logger should send a message
  to the web service if it throws an execption::

    [Test]
    public void analyze_LoggerThrows_CallsWebService()
    {
        var mockWebService = Substitute.For<IWebService>();
        var stubLogger = Substitute.For<ILogger>();
        stubLogger.When(logger => logger.LogError(Arg.Any<string>()))
                  .Do(info => { throw new Exception("fake exception"); });

        var analyzer = new LogAnalyzer(stubLogger, mockWebService);
        analyzer.MinNameLength = 10;
        analyzer.Analyze("Short.txt");

        mockWebService.Received()
            .Write(Arg.Is<string>(s => s.Contains("fake exeption")));
    }

  The assertion uses "argument-matching."

* A more complex argument matching if instead of a string,
  it is an `ErrorInfo` with `severity` and `message` properties::

    mockWebService.Received()
        .Write(Arg.Is<ErrorInfo>(info => info.Severity == 1000 &&
            info.Message.Contains("fake execption")));

* Perhaps a more readable test would compare `ErrorInfo` objects::

    var expected = new ErrorInfo(1000, "fake exception");
    mockWebService.Received().Write(expected);

  But the `Equals()` methods on `ErrorInfo` needs to have been
  implemented correctly.
