<p align="center">
  <img height="60" src="/images/metal_loop_circle.png" width="60" />
</p>

# Guidelines for Metal Component Developers

Some guidelines for writing maintainable and modular components.

## Metal

### State (and Config)

All attributes that a component may be passed should be listed in it's `STATE` configuration. While they are also available as `this.config[name]`, avoid referencing them there. Instead prefer `this[name]`, since props must be declared in `STATE` to be available, ensuring they are documented.

~~The exception to this rule is `children`, which is OK to reference from `this.config`.~~
`children` will be referenced with `this.children`([issue #118](https://github.com/metal/metal.js/issues/118))

```js
import Component from 'metal-jsx';
import Types from 'metal-state-validators';

class LabeledButton extends Component {
	renderLabel() {
		/**
		 * BAD: We should be able to see all of the accepted attributes in STATE,
		 * but we cannot here since config exposes any attribute passed to the component.
		 * In other words, don't expect anything other than children on the config object.
		 */
		return <span>{this.config.label}</span>;
	}

	render() {
		return (
			<div class="labeled-button">
				{this.renderLabel()}

				/**
				* GOOD: We know that buttonText is a something we can pass
				* to LabeledButton and that it should be a string.
				*/
				<Button onClick="onClick">{this.buttonText}</Button>
			</div>
		);
	}
}

LabeledButton.STATE = {
	buttonText: {
		validator: Types.string
	},

	onClick: {
		validator: Types.func
	}
};

export default LabeledButton;

```

`STATE` is self-documenting, and should be viewed as the component's public API. Metal makes no distinction between **public** and **private** attributes, in the way that React does with `props` and `state`.

For instances where we need a global component variable, but not be on the state object because it causes a re-render. We will pre-fix those attributes with an **underscore** and always declare their initial value in `created()`.

```js
class Component {
	created() {
		this._height = 0;
	}

	rendered() {
		this._height = this.element.height;
	}

	render() {
		//...
	}
}
```

If `this._height` was never re-declared to equal a new value, we would declare that on the state object with `readOnly: true` or we would declare it out of the context of the component entirely.

```js
const HEIGHT = '50px';

class Component {
//...
}
```

#### Organizing STATE Attributes

For our project specifically, we have written a helper called `createState()` that adds some sugar.

`createState()` accepts an object with 3 possible keys: `CONFIG`, `STATE`, and `STORE`. This allows us to visually organize the state object so that it's API is more visually apparent.

The `FollowMenu` component gives us a good example of this in action.

```js
const CONFIG = {
	id: Types.number,
	trigger: Types.any
};

const STATE = {
	open_: false
};

const STORE = {
	follow: Types.func,
	following: Types.bool,
	followingType: Types.number,
	notify: Types.func,
	notifyEmail: Types.func,
	notifying: Types.bool,
	notifyingEmail: Types.bool,
	schema: Types.string
};

FollowMenu.STATE = createState({CONFIG, STATE, STORE});

export default connect(
	(state, ownConfig) => {
		const {id} = ownConfig;
		let {schema} = ownConfig;

		if (isNumber(schema)) {
			schema = classNameIdToSchema(schema);
		}

		return {
			following: state.getIn([schema, id, 'data', 'following']),
			followingType: state.getIn([schema, id, 'data', 'followingType']),
			notifying: state.getIn([schema, id, 'data', 'notifying']),
			notifyingEmail: state.getIn([schema, id, 'data', 'notifyingEmail']),
			schema
		};
	},
	dispatch => bindActionCreators(
		{
			follow,
			notify,
			notifyEmail
		},
		dispatch
	)
)(FollowMenu);
```

There's a lot going on here, so let's break this down bit by bit.

```js
const CONFIG = {
	id: Types.number,
	trigger: Types.any
};
```

`CONFIG` is used to identify public attributes that the component is expecting to have passed in when it is instantiated. In this case when FollowMenu is rendered it will have an id and a trigger (this will be an element of component) passed into it. When you pass config a single value, it assumes you are passing it `validator`. If you need to pass it a `value`, you will need to pass the config an object that declares value as a key.

```js
const CONFIG = {
	id: {
		validator: Types.number,
		value: currentUser.classPK
	},
	trigger: Types.any
};
```

Use `STATE` do define attributes that will only be used internally. All `STATE` attributes should be suffixed with an underscore.


```js
const STATE = {
	open_: false
};
```

If a single value is passed to `STATE` it assumes you are passing it a `value`. This makes sense since we do not require private attributes to have a validator. If for some reason you still wanted to pass a validator or another state attribute setting, you can do so by passing it an object with the desired key.

```js
const STATE = {
	open_: {
		setter: value => !!value,
		value: false
	}
};
```

`STORE` is specific to Redux implementations. Use `STORE` for attributes that are passed in via the connect function. This will usually include actions as well as data that is retrieved from the store.

```js
const STORE = {
	follow: Types.func,
	following: Types.bool,
	followingType: Types.number,
	notify: Types.func,
	notifyEmail: Types.func,
	notifying: Types.bool,
	notifyingEmail: Types.bool,
	schema: Types.string
};
```

#### Naming Event Handlers

Event Handlers that are declared in the STATE as part of the public API should be named `on[descriptor][eventName]`.

```js
PostHeader.STATE = {
	onDateClick: {
		validator: Types.func
	},

	onLocationClick: {
		validator: Types.func
	}
}
```

In the case of a component having only one instance of a given event type (ie. button or checkbox), the descriptor can be left out.

```js
Button.STATE = {
	onClick: {
		validator: Types.func
	}
}
```

```js
Checkbox.STATE = {
	onChange: {
		validator: Types.func
	}
}
```

Functions that are declared on the component object and will be passed into an element or component attribute should follow a different naming convention. In order to keep them from being confused with event handlers that are passed into the component they will not begin with `on` instead they will follow the following pattern:

`handle[descriptor][event]`

```js
class Post extends Component {
	created() {
		bindAll(
			this,
			'handleDateClick',
			'handleLocationClick'
		);
	}

	handleDate() {
		// do something.
	}

	handleLocationClick(event) {
		// do something with the event object.
	}

	render() {
		const {handleDateClick, handleLocationClick} = this;

		return (
			<div>
				<PostHeader onDateClick={handleDateClick} onLocationClick={handleLocationClick} />
			</div>
		);
	}
}
```
In the case of `handle` type events, an `event` name is not always required. Event names are only required if the method uses the `event` object such as `handleLocationClick` which is seen above.


If using a method in multiple places, it is best that the descriptor would describe the action

```js
class Form extends Component {
	created() {
		this.handleMinimize = this.handleMinimize.bind(this);
	}

	handleMinimize() {
		// do something.
	}

	render() {
		const {handleMinimize} = this;

		return (
			<div>
				<Button onClick={handleMinimize} />

				<div>
					<span onClick={handleMinimize} />
				</div>
			</div>
		);
	}
}
```
#### Naming Functions that Return Renderable HTML

Functions that return html that can be rendered should be prefixed by `render`.

```
render[Descriptor]() {
	
}
```

### Destructuring Functions on the Component Object

Metal does not distinguish between functions declared on the component object and attributes passed into the component. This allows us to destructure functions alongside attributes. This is fine to do as long as the function that has been destructured is not being called in the current component but rather being passed throught to another component or element.

For example, `handleDateClick` and `handleLocationClick` may be destructured, but `renderPostContent()` should be called direclty off `this`.

```js
class Post extends Component {
	created() {
		bindAll(
			this,
			'handleDateClick',
			'handleLocationClick',
			'renderPostContent'
		);
	}

	handleDateClick() {
		// do something.
	}

	handleLocationClick() {
		// do something.
	}

	renderPostContent() {
		// do something.
	}

	render() {
		const {handleDateClick, handleLocationClick} = this;

		return (
			<div>
				<PostHeader onDateClick={handleDateClick} onLocationClick={handleLocationClick} />

				{this.renderPostContent()}
			</div>
		);
	}
}
```

### When to create a folder for component organization.

In the instance were a group of components are so related and interdependent that you would want to put them in the same file, create a folder and add them there. This is preferred over having multiple components in a single file because it keeps the line count of files from growing too large.

One instance where this should be implemented is Radio and Radio Group. These two components will always be used together, so it makes sense to let their file structure reflect that.

```
- Components/
	- radio-group/
		- __tests__/
			- Option.js
			- RadioGroup.js
		- index.js
		- Option.js
		- RadioGroup.js
```

We will use an index in order to export all of the related components. We are also able to namespace the components as we export them. This allows us to avoid adding a namespace within the radio folder, while still namespacing it where it is used.

`/index.js`
```js
import Option from './Option';
import RadioGroup from './RadioGroup';

RadioGroup.Option = Option;

export default RadioGroup;
```

When we implement RadioGroup it will look like this:

```js
import RadioGroup from '../radio-group';

<RadioGroup checked={checked} name="testradio" onChange={handleRadioChange}>
	<RadioGroup.Option label="Option 1" value={0} />
	<RadioGroup.Option label="Option 2" value={1} />
	<RadioGroup.Option label="Option 3" value={2} />
</RadioGroup>
```

### .bind()

In order to make sure a function has it's `this` keyword set to the current component you should bind functions in the `created` lifecycle method.

If you are only binding one function use the defaul Javascript bind method.

```js
class MyComponent extends Component {
	created() {
		this.myFunction = this.myFunction.bind(this);
	}

	myFunction() {
		// Do stuff with "this".
	}

	render() {
		return <div>My Component</div>;
	}
}
```

If you are binding multiple functions in the creator use lodash's bindAll method.

```js
import {bindAll} from 'lodash';

class MyComponent extends Component {
	created() {
		bindAll(
			this,
			'myFunction',
			'mySecondFunction'
		);
	}

	myFunction() {
		// Do stuff with "this".
	}


	mySecondFunction() {
		// Do stuff with "this".
	}

	render() {
		return <div>My Component</div>;
	}
}
```