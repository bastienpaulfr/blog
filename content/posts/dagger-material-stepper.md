---
title: "Use Dagger Provider with Android Material Stepper"
date: 2019-01-24T15:03:38+02:00
draft: false
---

On an Android project, I am using [android-material-stepper](https://github.com/stepstone-tech/android-material-stepper) from StepStone. I am also using dependency injection with [dagger](https://google.github.io/dagger/).

I was facing the following problem : How to properly inject `Fragments` without the need to create a `DaggerComponent` for each of them ?

Providers are here to help. This is how I am doing

```kotlin
class StepFragmentBL @Inject constructor() :
 Fragment(), Step, BlView {

    @Inject
    lateinit var blPresenter: BlPresenter

    ...
}
```

This is one of the Fragment I am creating. We can see the `@Inject constructor` with `@Inject lateinit` var that shows that this object is injectable by dagger. It is also implementing `Step` to be used by `StepperLayout` and implements another interface.

```kotlin
class BlPresenter @Inject constructor

BlPresenter is a simple injectable class

@Singleton
@Component(modules = [(DeliveryModule::class)])
interface DeliveryComponents {

    fun inject(deliveryActivity: DeliveryActivity)
}
```

This is the component injecting my Activity

```kotlin
@Module
class DeliveryModule(private val activity: DeliveryActivity) {

    @Named("Activity")
    @Provides
    internal fun providesContext(): Context {
        return activity
    }

    @Provides
    fun providesFragmentManager(): FragmentManager {
        return activity.supportFragmentManager
    }
}
```

```kotlin
class DeliveryActivity : AppCompatActivity() {

    @Inject
    lateinit var deliveryPresenter: DeliveryPresenter

    private lateinit var deliveryComponents: DeliveryComponents

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_home)

        di()
        ui()
    }

    private fun di() {
        deliveryComponents = DaggerDeliveryComponents.builder()
                .deliveryModule(DeliveryModule(this))
                .build()
        deliveryComponents.inject(this)
    }

    private fun ui() {
        Timber.v(Info.getMethodName())
        stepperLayout.adapter = deliveryPresenter.stepperAdapter
    }
}
```

We are creating the component into the `Activity` here. Then, presenter for this activity is also injected

```kotlin
class DeliveryPresenter @Inject constructor() {

    @Inject
    lateinit var stepperAdapter: PickingStepperAdapter

}
```

We are arriving to a `StepperAdapter` instance, also injected

```kotlin
private const val NB_FRAG = 2

class PickingStepperAdapter @Inject constructor(
        fm: FragmentManager,
        @Named("Activity") context: Context) :
            AbstractFragmentStepAdapter(fm, context) {

    /**
    * Provider for Fragment
    */
    @Inject
    lateinit var blFragmentProvider: Provider<StepFragmentBL>

    /**
    * Provider for Fragment
    */
    @Inject
    lateinit var pdtFragmentProvider: Provider<StepFragmentPdt>

    override fun getCount(): Int {
        return NB_FRAG
    }

    override fun createStep(position: Int): Step {
        Timber.v("${Info.getMethodName()} $position")

        return when (position) {
            0 -> blFragmentProvider.get()
            else -> pdtFragmentProvider.get()
        }
    }
}
```

Here, we are just using

```kotlin
@Inject
    lateinit var blFragmentProvider: Provider<StepFragmentBL>
```

to tell dagger to create the code that will provide an instance of fragment each time `blFragmentProvider.get()` will be called. Indeed, this is called here

```kotlin
return when (position) {
            0 -> blFragmentProvider.get()
            else -> pdtFragmentProvider.get()
        }
```

Like this, dagger is injecting fragment for us, and we don’t have to care about objects creation
