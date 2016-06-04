##Realm Java
你需要知道的所有信息都在这里
<https://realm.io/cn/docs/java/latest>

Realm Java 让你能够高效地编写 app 的模型层代码，保证你的数据被安全、快速地存储。参考下列示例来开始你的 Realm 之旅：
	
	// Define you model class by extending RealmObject
	public class Dog extends RealmObject {
	    private String name;
	    private int age;
	
	    // ... Generated getters and setters ...
	}
	
	public class Person extends RealmObject {
	    @PrimaryKey
	    private long id;
	    private String name;
	    private RealmList<Dog> dogs; // Declare one-to-many relationships 
	
	    public Person(long id, String name) {
	        this.id = id;
	        this.name = name;
	    }
	
	    // ... Generated getters and setters ...
	}
	
	// Use them like regular java objects
	Dog dog = new Dog();
	dog.setName("Rex");
	dog.setAge(1);
	
	// Create a RealmConfiguration that saves the Realm file in the app's "files" directory.
	RealmConfiguration realmConfig = new RealmConfiguration.Builder(context).build();
	Realm.setDefaultConfiguration(realmConfig);
	
	// Get a Realm instance for this thread
	Realm realm = Realm.getDefaultInstance();
	
	// Query Realm for all dogs younger than 2 years old
	final RealmResults<Dog> puppies = realm.where(Dog.class).lessThan("age", 2).findAll();
	puppies.size(); // => 0 because no dogs have been added to the Realm yet
	
	// Persist your data in a transaction
	realm.beginTransaction();
	final managedDog = realm.copyToRealm(dog); // Persist unmanaged objects
	Person person = realm.createObject(Person); // Create managed objects directly
	person.getDogs().add(managedDog);
	realm.commitTransaction();
	
	// Listeners will be notified when data changes
	puppies.addChangeListener(new RealmChangeListener<RealmResults<Dog>>() {
	    @Override
	    public void onChange(RealmResults<Dog> results) {
	        // Query results are updated in real time
	        puppies.size(); // => 1
	    }
	});
	
	// Asynchronously update objects on a background thread
	realm.executeTransactionAsync(new Realm.Transaction() {
	    @Override
	    public void execute(Realm bgRealm) {
	        Dog dog = bgRealm.where(Dog.class).equals("age", 1).findFirst();
	        dog.setAge(3);
	    }
	}, new Realm.Transaction.OnSuccess() {
	    @Override
	    public void onSuccess() {
	    	// Original queries and Realm objects are automatically updated.
	    	puppies.size(); // => 0 because there are no more puppies younger than 2 years old
	    	managedDog.getAge();   // => 3 the dogs age is updated
	    }
	});