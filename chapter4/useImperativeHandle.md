## useImperativeHandle

useImperativeHandle用于自定义暴露给父组件的ref属性。需要配合forwardRef一起使用。

```
function Example(props, ref) {
    const inputRef = useRef();
    useImperativeHandle(ref, () => ({
        focus: () => {
            inputRef.current.focus();
        }
    }));
    return <input ref={inputRef} />;
}

export default forwardRef(Example);

```

```
class App extends Component {
  constructor(props){
      super(props);
      this.inputRef = createRef()
  }
  
  render() {
    return (
        <>
            <Example ref={this.inputRef}/>
            <button onClick={() => {this.inputRef.current.focus()}}>Click</button>
        </>
    );
  }
}

```