# An Introduction to MVP on Android
It's already proven<sup>1</sup> that using the default architecture provided by the Android SDK isn't a good choice for building complex Android applications which are meant to be **testable**, **maintainable** and always kept **up to date**. This article is going to introduce **MVP** (Model View Presenter) pattern as a better approach to build your applications and help you easily develop better Android applications based on MVP pattern using [**EasyMVP** library](https://github.com/6thsolution/EasyMVP).
So first let's observe MVP on Android and then we'll dig into it's implementaion with the help of EasyMVP library.

  * [What Is MVP](#what-is-mvp)
  * [MVP On Android](#mvp-on-android)
	  * [Why?](#why)
	  * [How MVP solves these problems?](#how-mvp-solves-these-problems)
  * [Starting Guide](#starting-guide-using-easymvp-to-implement-mvp-on-android)
	  * [MvpView](#mvpview-code:)
	  * [MvpPresenter](#mvppresenter-code:)
	  * [MvpActivity](#mvpactivity-code:)
  * [Best Practice](#best-practice-easymvp-and-clean-architecture)
  * [How does EasyMVP work?](#how-does-easymvp-work?)


## What Is MVP?
[MVP](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter) stands for Model View Presenter and is an **architectural pattern** that separates an application into three layers:

 - **Model:** holds the business logic of our application and is responsible for all data interactions in the application.
 - **View:** a passive interface that is responsible for all the UI/UX related things, shows data to the user and reacts to the input events 
 - **Presenter:** acts as the middle man, provides view with the data from model and defines reactions to user input which view forwards to it.
##MVP On Android
As default android architecture wasn't developed with good attention to **separation of concerns**, MVP is used to separate UI/UX (Activity, Fragment, ...) from I/O and application's business logic. But how this separation helps us?
###Why? 
In default Android application architecture, activities (or their replacements, like fragments and ...) are **god objects** and extremely **coupled** to both UI interface and business logic or data manipulations. In this god object, literally *everything is connected to everything* and the code is **over complicated.** So changing a little part of the code requires to update the entire code and takes a lot of effort. Also because different parts are connected, no part is **reusable** or **testable**.

Moreover, handling the background tasks in this code is really a pain in the neck. Apart from having to handle numerous edge cases for **configuration changes**, the probability of having **memory leaks** grows exponentially as your code gets more and more complex.
Adding another parts and features to this code is also extremely sorrowful because you'll probably need to check most of the code and update it!

Finally you'll find multiple bugs which are not at all easy to even find, let alone **debugging** and solving them. 
###How MVP solves these problems?
With separating the application into three main layers, the **view** which is normally the activity or fragment is only responsible for the UI/UX part, like showing data to the user, handling animations, forwarding user input to presenter and navigating between screens.  
The view is also responsible for handling lifecycle events for presenter and **saving/restoring the presenter** when needed (for example in the onResume/onPause methods of an activity).
 
The **model** layer is only responsible for data interactions and all the **business logic** and data manipulations and doesn't know anything about the application's UI and doesn't even have a reference to the presenter or the view.

The **presenter** layer is a new part which isn't available in the plain old android architecture. This layer almost acts as the brain of the application and **decides** everything about the application's behavior. Presenter in android is a **pure java** class that we're able to do **unit tests** on it. It fills the view layer with the data that it retrieves from the model layer.

With this separation, there's no god object and each layer follows this connection rule:
**Picture of MVP design: Model <-> View <-> Presenter**

This way our code isn't complicated, different parts can be easily **maintained**, **tested**, **debugged** and **reused**. As view saves and restores presenter and handles the presenter's lifecycle, the **background tasks** are now performed in presenter without concerning about the lifecycle events and configuration changes. They are saved/restored automatically while we're sure there'll be **no memory leak**. 

**Abstraction** of these layers helps us make sure that no layer knows anything about the implementation of any other layer and each part of the application is completely **decoupled**.

## **Starting Guide**: Using EasyMVP to implement MVP on Android
In this example we're going to fetch data from a remote server and show it in our activity as a `ListView`.  We'll use **EasyMVP** to implement MVP pattern in our approach. So as mentioned above, the `activity` is going to play the **view** role, the **presenter** is going to be a  java class and the **model** retrieves information from the remote server and provides a `RxJava Observable` for the presenter to use. 

These classes show how the above idea can be easily implemented using *only a few annotations* from **EasyMVP** :


####**MvpView** code:

```java
public interface MvpView {
    void showData(List<String> items);
    void showLoading();
    void showError();
}
```

####**MvpPresenter** code:

```java
public class MvpPresenter extends RxPresenter<MvpView> {

    private List<String> items;

    private void reloadData() {
        this.items = null;
        loadData();
    }

    private void loadData() {

        /**
         * Checking nullity because {@link Presenter#getView()} is {@link javax.annotation.Nullable}
         */
        if (getView() != null) {
            getView().showLoading();
        } else {
            return;
        }

        /**
         * Loading our data from a Web API using an {@link rx.Observable} from RxJava
         */
        Subscription subscription = API.getDataObservable()
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(
                        items -> {
                            // Showing data when items are retrieved
                            this.items = items;
                            if (getView() != null) {
                                getView().showData(items);
                            }
                        },
                        throwable -> {
                            // Printing the log and showing error
                            throwable.printStackTrace();
                            if (getView() != null) {
                                getView().showError();
                            }
                        }
                );

        /**
         * Adding the {@link Subscription} to the presenter so
         * it will automatically get unsubscribed at {@link RxPresenter#onDestroyed()}
         */
        addSubscription(subscription);

    }

    @Override
    public void onViewAttached(MvpView view) {
        super.onViewAttached(view);
        // Checking if the data has been loaded before
        if (items == null) {
            loadData();
        }
    }
}

```

####**MvpActivity** code:

```java
@ActivityView(presenter = MvpPresenter.class, layout = R.layout.activity_mvp)
public class MvpActivity extends AppCompatActivity implements MvpView {

    private ListView listView;
    private LinearLayout errorView;
    private ProgressBar progressBar;

    private ArrayAdapter<String> adapter = new ArrayAdapter<>(this, R.layout.item);

    @Presenter
    MvpPresenter presenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        initializeViews();
        listView.setAdapter(adapter);

        findViewById(R.id.reloadbutton).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                presenter.reloadData();
            }
        });

    }

    private void initializeViews() {
        listView = (ListView) findViewById(R.id.listview);
        errorView = (LinearLayout) findViewById(R.id.errorview);
        progressBar = (ProgressBar) findViewById(R.id.progressbar);
    }

    @Override
    public void showData(List<String> items) {

        adapter.clear();
        adapter.addAll(items);

        errorView.setVisibility(View.GONE);
        progressBar.setVisibility(View.GONE);
        listView.setVisibility(View.VISIBLE);

    }

    @Override
    public void showLoading() {
        listView.setVisibility(View.GONE);
        errorView.setVisibility(View.GONE);
        progressBar.setVisibility(View.VISIBLE);
    }

    @Override
    public void showError() {
        listView.setVisibility(View.GONE);
        progressBar.setVisibility(View.GONE);
        errorView.setVisibility(View.VISIBLE);
    }

}
```

As you might have mentioned, we use `@ActivityView` to attach the presenter to the activity and set its layout id and `@Presenter` in the activity to define the presenter field. The‍ ‍‍‍‍‍‍‍‍`activity` implements the `view interface` so that we're sure its completely decoupled from `presenter`.

In presenter's `OnViewAttached` method we check if we should load the data because it will get called everytime the view becomes on screen such as in `onCreate` method of the activity (more information can be found [here](https://github.com/6thsolution/EasyMVP/)).

Finally activity (view) forwards any button click to presenter so the user input is also handled in the presenter and there's **no business logic in view**.

EasyMVP does all the hard work here, like saving/restoring presenter's stMVate in and preventing any **memory leaks** by releasing the view in presenter in lifecycle events. You can proceed to the next example to see all EasyMVP features.

## Best practice: EasyMVP and Clean Architecture
**Comming soon:** we're publishing an example to observer all features of EasyMVP in developing a feature rich Android application based on Clean Architecture and MVP pattern.
## How does EasyMVP work?
**Comming soon**: we're going to publish an article on how EasyMVP works to help us build better Android applications. 

#### Refrences
1: There are a lot of articles out there that show why MVP should be used in Android applications, some of them are listed here:

 - https://antonioleiva.com/mvp-android/
 - http://engineering.remind.com/android-code-that-scales/
 - https://www.ackee.cz/en/blog/an-introduction-to-mvp-on-android/
 - https://code.tutsplus.com/tutorials/an-introduction-to-model-view-presenter-on-android--cms-26162
