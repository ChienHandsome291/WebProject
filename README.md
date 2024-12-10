-slide.js
import { createSlice, createAsyncThunk } from "@reduxjs/toolkit";
import axios from "axios";

export const createReusableSlice = ({ name, apiUrl }) => {
  
  const fetchItems = createAsyncThunk(`${name}/fetchItems`, async () => {
    const response = await axios.get(apiUrl);
    return response.data;
  });

  const addItem = createAsyncThunk(`${name}/addItem`, async (newItem) => {
    const response = await axios.post(apiUrl, newItem);
    return response.data;
  });

  const updateItem = createAsyncThunk(`${name}/updateItem`, async (updatedItem) => {
    const response = await axios.put(`${apiUrl}/${updatedItem.id}`, updatedItem);
    return response.data;
  });

  const deleteItem = createAsyncThunk(`${name}/deleteItem`, async (id) => {
    await axios.delete(`${apiUrl}/${id}`);
    return id;
  });

  const slice = createSlice({
    name,
    initialState: {
      items: [],
      loading: false,
      error: null,
    },
    reducers: {},
    extraReducers: (builder) => {
      builder
        .addCase(fetchItems.pending, (state) => {
          state.loading = true;
          state.error = null;
        })
        .addCase(fetchItems.fulfilled, (state, action) => {
          state.loading = false;
          state.items = action.payload;
        })
        .addCase(fetchItems.rejected, (state, action) => {
          state.loading = false;
          state.error = action.error.message;
        })
        .addCase(addItem.fulfilled, (state, action) => {
          state.items.push(action.payload);
        })
        .addCase(updateItem.fulfilled, (state, action) => {
          const index = state.items.findIndex((item) => item.id === action.payload.id);
          if (index !== -1) {
            state.items[index] = action.payload;
          }
        })
        .addCase(deleteItem.fulfilled, (state, action) => {
          state.items = state.items.filter((item) => item.id !== action.payload);
        });
    },
  });

  return {
    slice: slice.reducer,
    actions: {
      fetchItems,
      addItem,
      updateItem,
      deleteItem,
    },
  };
};



-store.js
import { configureStore } from "@reduxjs/toolkit";
import { createReusableSlice } from "./slice.js";

const postsSlice = createReusableSlice({
  name: "Contents",
  apiUrl: "https://6715bd7d33bc2bfe40bafa1c.mockapi.io/api/v1/Contents",
});

export const store = configureStore({
  reducer: {
    Contents : postsSlice.slice,
  }
});

export const { fetchItems, addItem, updateItem, deleteItem } = postsSlice.actions;

-List.tsx
import { FlatList, Text, TouchableOpacity, StyleSheet, SafeAreaView, View, Alert } from 'react-native';
import { useEffect } from 'react';
import { useSelector, useDispatch } from 'react-redux';
import { fetchItems, deleteItem } from '../reduxToolKit/store';

const Item = ({ item, navigation }) => {
  const dispatch = useDispatch();
  const handleDelete = (id) => {
    dispatch(deleteItem(id))
  };

  return (
    <View style={styles.itemContainer}>
      <View style={styles.contentContainer}>
        <Text style={styles.itemName}>
          {item.name}
        </Text>
        <Text style={styles.itemDesc}>
          {item.description}
        </Text>
      </View>

      <View style={styles.buttonContainer}>
        <TouchableOpacity style={styles.button} onPress={() => navigation.navigate('UpdateAndAdd', { item })}>
          <Text style={styles.buttonText}>Update</Text>
        </TouchableOpacity>

        <TouchableOpacity style={styles.button} onPress={() => handleDelete(item.id)}>
          <Text style={styles.buttonText}>Delete</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
};

const List = ({ navigation }) => {
  const { items, isLoading, error } = useSelector((state) => state.Contents);
  const dispatch = useDispatch();
  useEffect(() => {
    dispatch(fetchItems());
  }, [dispatch]);

  return (
    <SafeAreaView style={styles.container}>
      <TouchableOpacity
        onPress={() => navigation.navigate('UpdateAndAdd')}
        style={styles.addButton}>
        <Text style={styles.plusIcon}>+</Text>
      </TouchableOpacity>

      <FlatList
        style={styles.list}
        keyExtractor={(i) => i.id.toString()}
        renderItem={({ item }) => <Item item={item} navigation={navigation} />}
        data={items}
        numColumns={1}
      />
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    position: 'relative',
  },
  list: {
    flex: 1,
    padding: 4,
  },
  itemContainer: {
    marginVertical: 5,
    backgroundColor: '#abdbe3',
    width: '100%',
    flexDirection: 'row',
    padding: 4,
    borderRadius: 8,
  },
  itemName: {
    fontSize: 18,
    fontWeight: 'bold',
  },
  itemDesc: {
    fontSize: 14,
  },
  contentContainer: {
    width: '70%',
  },
  buttonContainer: {
    flexDirection: 'column',
    width: '30%',
    alignItems: 'center',
  },
  buttonText: {
    color: '#FFF',
  },
  button: {
    backgroundColor: '#e28743',
    width: '100%',
    margin: 5,
    padding: 4,
    alignItems: 'center',
    borderRadius: 8,
  },

  addButton: {
    position: 'absolute',
    zIndex: 2,
    bottom: 50,
    right: 50,
    borderRadius: '50%',
    backgroundColor: '#FFF',
    display: 'flex',
    justifyContent: 'center',
    alignItems: 'center',
    width: 40,
    height: 40,
  },

  plusIcon: {
    fontSize: 24, width: 20, height: 20,
    transform: [
      { translateX: 2 },
      { translateY: -8 }, 
    ],
  },
});

export default List;



-UpdateAndAdd.jsx
import {View, SafeAreaView, Text, TextInput, TouchableOpacity, StyleSheet, KeyboardAvoidingView, Keyboard, Alert} from 'react-native';
import { useEffect, useState } from 'react';
import { useDispatch } from 'react-redux';
import { updateItem, addItem } from '../reduxToolKit/store';

const UpdateAndAdd = ({ route, navigation }) => {
  const [inputName, setInputName] = useState('');
  const [inputDesc, setInputDesc] = useState('');
  const [idEdit, setIdEdit] = useState(null);

  const dispatch = useDispatch();

  const resetState = () => {
    setIdEdit(null);
    setInputName('');
    setInputDesc('');
  };

  useEffect(() => {
    if (route.params?.item) {
      const { item } = route.params;
      setIdEdit(item.id);
      setInputName(item.name);
      setInputDesc(item.description);
    }

    return resetState;
  }, [route.params]);

  const handleUpdateAndAdd = () => {
    if (!inputName.trim() || !inputDesc.trim()) {
      Alert.alert('Thông báo', 'Các trường không được để trống.');
      return;
    }

    try {
      if (idEdit) {
        dispatch(updateItem({ id: idEdit, name: inputName, description: inputDesc }));
      } else {
        dispatch(addItem({ name: inputName, description: inputDesc }));
      }
      resetState();
      Keyboard.dismiss();
      navigation.goBack(); 
    } catch (error) {
      Alert.alert('Lỗi', 'Đã xảy ra lỗi khi cập nhật hoặc thêm mục.');
    }
  };

  return (
    <SafeAreaView style={styles.container}>
      <KeyboardAvoidingView behavior="padding" style={{ flex: 1 }}>
        <Text style={styles.headerText}>{idEdit ? 'Cập nhật mục' : 'Thêm mục mới'}</Text>

        <TextInput
          value={inputName}
          onChangeText={setInputName}
          style={styles.input}
          placeholder="Nhập tên mới"
        />

        <TextInput
          value={inputDesc}
          onChangeText={setInputDesc}
          style={styles.input}
          placeholder="Nhập nội dung mới"
          multiline
        />

        <TouchableOpacity style={styles.button} onPress={handleUpdateAndAdd}>
          <Text style={styles.buttonText}>{idEdit ? 'Cập nhật' : 'Thêm'}</Text>
        </TouchableOpacity>
      </KeyboardAvoidingView>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    paddingHorizontal: 16,
    backgroundColor: '#f5f5f5',
  },
  headerText: {
    fontSize: 24,
    fontWeight: 'bold',
    marginBottom: 20,
    textAlign: 'center',
    color: '#333',
  },
  input: {
    padding: 10,
    marginVertical: 8,
    width: '100%',
    borderWidth: 1,
    borderRadius: 16,
    borderColor: '#ccc',
    backgroundColor: '#fff',
    fontSize: 16,
  },
  button: {
    marginTop: 20,
    padding: 15,
    width: '100%',
    backgroundColor: 'blue',
    borderRadius: 25,
  },
  buttonText: {
    fontSize: 18,
    color: 'white',
    textAlign: 'center',
  },
});

export default UpdateAndAdd;



-Start.jsx
import {View, SafeAreaView, Text, TextInput, TouchableOpacity, StyleSheet, KeyboardAvoidingView, Keyboard, Alert} from 'react-native';
import { useEffect, useState } from 'react';
import { useDispatch } from 'react-redux';
import { updateItem, addItem } from '../reduxToolKit/store';

const UpdateAndAdd = ({ route, navigation }) => {
  const [inputName, setInputName] = useState('');
  const [inputDesc, setInputDesc] = useState('');
  const [idEdit, setIdEdit] = useState(null);

  const dispatch = useDispatch();

  const resetState = () => {
    setIdEdit(null);
    setInputName('');
    setInputDesc('');
  };

  useEffect(() => {
    if (route.params?.item) {
      const { item } = route.params;
      setIdEdit(item.id);
      setInputName(item.name);
      setInputDesc(item.description);
    }

    return resetState;
  }, [route.params]);

  const handleUpdateAndAdd = () => {
    if (!inputName.trim() || !inputDesc.trim()) {
      Alert.alert('Thông báo', 'Các trường không được để trống.');
      return;
    }

    try {
      if (idEdit) {
        dispatch(updateItem({ id: idEdit, name: inputName, description: inputDesc }));
      } else {
        dispatch(addItem({ name: inputName, description: inputDesc }));
      }
      resetState();
      Keyboard.dismiss();
      navigation.goBack(); 
    } catch (error) {
      Alert.alert('Lỗi', 'Đã xảy ra lỗi khi cập nhật hoặc thêm mục.');
    }
  };

  return (
    <SafeAreaView style={styles.container}>
      <KeyboardAvoidingView behavior="padding" style={{ flex: 1 }}>
        <Text style={styles.headerText}>{idEdit ? 'Cập nhật mục' : 'Thêm mục mới'}</Text>

        <TextInput
          value={inputName}
          onChangeText={setInputName}
          style={styles.input}
          placeholder="Nhập tên mới"
        />

        <TextInput
          value={inputDesc}
          onChangeText={setInputDesc}
          style={styles.input}
          placeholder="Nhập nội dung mới"
          multiline
        />

        <TouchableOpacity style={styles.button} onPress={handleUpdateAndAdd}>
          <Text style={styles.buttonText}>{idEdit ? 'Cập nhật' : 'Thêm'}</Text>
        </TouchableOpacity>
      </KeyboardAvoidingView>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    paddingHorizontal: 16,
    backgroundColor: '#f5f5f5',
  },
  headerText: {
    fontSize: 24,
    fontWeight: 'bold',
    marginBottom: 20,
    textAlign: 'center',
    color: '#333',
  },
  input: {
    padding: 10,
    marginVertical: 8,
    width: '100%',
    borderWidth: 1,
    borderRadius: 16,
    borderColor: '#ccc',
    backgroundColor: '#fff',
    fontSize: 16,
  },
  button: {
    marginTop: 20,
    padding: 15,
    width: '100%',
    backgroundColor: 'blue',
    borderRadius: 25,
  },
  buttonText: {
    fontSize: 18,
    color: 'white',
    textAlign: 'center',
  },
});

export default UpdateAndAdd;



-NavigationContainer.jsx
import { NavigationContainer } from "@react-navigation/native";
import { createBottomTabNavigator } from "@react-navigation/bottom-tabs";
import { createStackNavigator } from "@react-navigation/stack";
import { Ionicons } from "@expo/vector-icons";
import List from "../components/List.jsx";
import UpdateAndAdd from "../components/UpdateAndAdd.jsx";
import Start from '../components/Start.jsx';

const Stack = createStackNavigator();
const Tabs = createBottomTabNavigator();

const AppTabs = () => {
  const getTabIcon = (routeName) => {
    switch (routeName) {
      case "List":
        return "home-outline";
      case "UpdateAndAdd":
        return "settings-outline";
      default:
        return "help-circle-outline";
    }
  };

  return (
    <Tabs.Navigator
      screenOptions={({ route }) => ({
        tabBarIcon: ({ color, size }) => (
          <Ionicons name={getTabIcon(route.name)} size={size} color={color} />
        ),
        tabBarActiveTintColor: "#007AFF", 
        tabBarInactiveTintColor: "#8E8E93", 
        tabBarStyle: {
          backgroundColor: "#F8F8F8",
          borderTopWidth: 0.5,
          borderTopColor: "#E2E2E2",
        },
        headerShown: false,
      })}
    >
      <Tabs.Screen name="List" component={List} options={{ title: "Danh sách" }} />
      <Tabs.Screen
        name="UpdateAndAdd"
        component={UpdateAndAdd}
        options={{ title: "Cập nhật/Thêm" }}
      />
    </Tabs.Navigator>
  );
};

const AppNavigation = () => {
  return (
    <NavigationContainer>
      <Stack.Navigator screenOptions={{ headerShown: false }}>
        <Stack.Screen name='Start' component={Start} />
        <Stack.Screen name="Tabs" component={AppTabs} />
        <Stack.Screen
          name="UpdateAndAdd"
          component={UpdateAndAdd}
          options={{
            headerShown: true, 
            headerTitle: "Thêm hoặc Cập nhật", 
            headerStyle: {
              backgroundColor: "#007AFF",
            },
            headerTintColor: "#FFF", 
          }}
        />
      </Stack.Navigator>
    </NavigationContainer>
  );
};

export default AppNavigation;



-package.json
{
  "dependencies": {
    "axios": "*",
    "redux": "*",
    "react-redux": "9.0.0",
    "@types/react": "18.2.41",
    "@reduxjs/toolkit": "*",
    "@types/react-dom": "18.2.17",
    "@expo/vector-icons": "^14.0.3",
    "react-native-screens": "3.31.1",
    "@react-navigation/stack": "*",
    "@react-navigation/native": "*",
    "react-native-gesture-handler": "~2.16.1",
    "@react-navigation/bottom-tabs": "*",
    "react-native-safe-area-context": "4.10.5",
    "@expo/vector-icons/SimpleLineIcons": "^14.0.3"
  }
}
